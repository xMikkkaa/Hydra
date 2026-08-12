# HYDRA (High-Yield Dynamic Render Affinity)

Kernel-level game thread optimizer for Android — automatic nice boost + big core affinity for rendering threads.

## Overview
- **What it does**: Applies a nice boost (default -10), pins specific render threads to Big/Prime cores, and enforces `uclamp_min` frequency boosting.
- **Hybrid Matching**: Supports both static fallback patterns (`RenderThread`, etc.) and real-time dynamic patterns injected via `/proc/sys/kernel/hydra_thread_patterns`.
- **On-demand**: Activated by writing a game PID to `/proc/sys/kernel/hydra_pid`. Zero overhead when idle.
- **Smart Mode & Topology**: Adaptive thermal-aware cpumask management that instantly responds to CPU hotplug events.
- **Isolated**: Lives entirely in `hydra.c` and `hydra.h`, never modifying core scheduler files (`fair.c`, `core.c`, etc.).

## How It Works

```mermaid
flowchart LR
    A["Game process"] --> B["Aozora Kernel Manager"]
    B -->|"Write PID & Dynamic Patterns"| C["/proc/sys/kernel/hydra_* sysctls"]
    C --> D["hydra.c sysctl handler"]
    D --> E["hydra_optimize_threads()"]
    E --> F{"Match Static &
Dynamic Patterns"}
    F --> G["set_user_nice()"]
    F --> H["set_cpus_allowed_ptr()
→ Big/Prime Cores"]
    F --> U["uclamp_min Boost"]
    I["sched_process_fork tracepoint"] --> E
    J["sched_process_exit tracepoint"] --> K["hydra_revert_all_threads()"]
    L["CPU Hotplug (cpuhp)"] --> E
    S["Smart worker (500ms, mode 2)"] --> M["Thermal Sweep (for_each_cpu_and)"]
    M --> H
```

When a game PID is written to `hydra_pid`, HYDRA scans the process and its threads, matching their names against known rendering thread patterns. Matched threads are boosted via `set_user_nice()` and pinned to big cores via `set_cpus_allowed_ptr()`. HYDRA hooks into scheduler tracepoints to catch newly spawned threads (`sched_process_fork`) and handle cleanup upon exit (`sched_process_exit`). A dedicated worker periodically checks task utilization and thermals to adaptively manage CPU affinity when Smart Mode is active.

## Requirements
- Android kernel source tree (msm-4.9, msm-4.14, msm-4.19, msm-5.4, or msm-5.10)
- ARM64 architecture (designed for SDM845, works on other Qualcomm SoCs with multi-cluster topologies like big.LITTLE or 1+3+4 Prime/big/LITTLE)
- Root access
- `CONFIG_SCHED_HYDRA=y` in your defconfig
- Recommended: [Aozora Kernel Manager](https://github.com/xMikkkaa/Aozora-Kernel-Manager) for automated game detection

## Installation
1. Clone this repository.
2. Apply the patch to your kernel tree:
   ```bash
   cd your-kernel-tree
   git apply path/to/patches/testing/0001-android-msm-hydra-0.9.patch
   ```
   > **Note**: If `git apply` fails due to context mismatch in `init/Kconfig` or `kernel/sched/Makefile` on custom kernels, you can safely skip patching those two files and add them manually:
   > 1. In `kernel/sched/Makefile`, append this line anywhere: `obj-$(CONFIG_SCHED_HYDRA) += hydra.o`
   > 2. In `init/Kconfig`, paste the `config SCHED_HYDRA` block (found inside the patch file) anywhere before the last `endmenu`.
3. Add to your defconfig: `CONFIG_SCHED_HYDRA=y`
4. Build the kernel normally.
5. Flash the kernel.
6. Install Aozora Kernel Manager for automatic game management.

## Userspace Integration — Aozora Kernel Manager
HYDRA is designed to work seamlessly with [Aozora Kernel Manager](https://github.com/xMikkkaa/Aozora-Kernel-Manager). Aozora features a built-in Rust daemon that:
- Detects foreground game applications.
- Writes the game PID to `/proc/sys/kernel/hydra_pid`.
- Writes `0` when the game exits (the kernel also has an auto-revert via the exit hook as a fallback).
- Supports per-game profiles that map to HYDRA sysctls.

### Manual Usage
```bash
# Manual activation
echo $GAME_PID > /proc/sys/kernel/hydra_pid

# Check status
cat /proc/sys/kernel/hydra_stats

# Deactivate
echo 0 > /proc/sys/kernel/hydra_pid
```

## Tunables
HYDRA provides several sysctl interfaces to tune its behavior.

Example:
```bash
sudo sysctl -w kernel.hydra_enable=1
```

All tunables are located under `/proc/sys/kernel/`:

| Name | Default | Range | Description |
|------|---------|-------|-------------|
| `hydra_enable` | `1` | `0-1` | Global toggle. Uses static key. Off = full revert. |
| `hydra_pid` | `0` | `0` / valid PID | Target game PID. `0` = idle. Changing value reverts old + optimizes new. |
| `hydra_nice` | `-10` | `-15` to `-1` | Nice value applied to matched threads. |
| `hydra_smart_mode` | `2` | `0-2` | `0`=Off (revert, state kept), `1`=Hard Pin, `2`=Smart adaptive. |
| `hydra_throttle_freq` | `1400000` | kHz | If big core freq falls below this, it is considered throttled and the cpumask is released. |
| `hydra_heavy_util` | `300` | `0-1024` | Utilization above this pins the thread to big cores. |
| `hydra_light_util` | `100` | `0-1024` | Utilization below this restores the original cpumask. |
| `hydra_stats` | *(read-only)* | - | Shows: tracked threads, mode, throttle state, active PID, version. |
| `hydra_cluster_depth` | `0` | `0` to `max_clusters-1` | Number of top clusters to pin (0 = auto, pins all except little). |
| `hydra_thread_patterns`| `""` (empty) | String | Comma-separated list of thread names to match dynamically. |
| `hydra_uclamp_min` | `0` | `0-1024` | uclamp_min value to boost matched threads (0 = disable). |

**IMPORTANT**: `hydra_light_util` must be ≤ `hydra_heavy_util`. The gap between these two values acts as a hysteresis zone where the thread maintains its current state, intentionally preventing cpumask thrashing.

## Smart Mode Explained
- **Mode 0 (Off)**: Reverts cpumask and nice values, but keeps tracking the state for when it is re-enabled.
- **Mode 1 (Hard Pin)**: Pins all matched threads to big cores unconditionally.
- **Mode 2 (Smart/Adaptive)**: A 500ms worker checks per-thread utilization and big core frequency:
  - `util > heavy_util` → pin to big cores.
  - `util < light_util` → restore original cpumask.
  - **Hysteresis zone** (between light and heavy util) → no change (prevents thrashing).
  - If big core freq < `throttle_freq` → all threads get original cpumask (thermal protection).

## Thread Matching
HYDRA uses a **hybrid matching architecture**:
1. **Static Patterns**: It first checks against a built-in list of common game threads (`RenderThread`, `UnityMain`, `UnityGfx`, `TaskGraph`, `GameThread`, `adreno`, `glthread`, `kgsl`, `ANGLE`, `FrameWorker`).
2. **Dynamic Patterns**: If the thread doesn't match the static list, it checks against dynamically configured patterns from `/proc/sys/kernel/hydra_thread_patterns`.

By default, the dynamic pattern sysctl is empty (`""`). Userspace daemons (like Aozora Kernel Manager) can safely inject new game-specific threads into the sysctl without accidentally overriding or deleting the static core patterns.

Matching uses `strnstr()` on the task comm. To add new patterns, simply write a comma-separated string to the sysctl interface.

## Security
- All sysctl writes require `CAP_SYS_NICE`.
- HYDRA only modifies nice values, uclamp, and cpumasks — no privilege escalation is possible.
- Thread manipulation happens in workqueue context, not interrupt or probe context.
- Identity validation (using TID + start_time) prevents accidental manipulation of the wrong threads (e.g., PID recycling).

## Known Limitations
- Big core detection uses `capacity_orig_of()`, requiring a properly configured CPU topology.
- Maximum of 64 tracked threads per game (`HYDRA_MAX_TRACKED_THREADS`).
- Smart mode polling occurs at 500ms intervals, which is not sub-frame granularity.

## Roadmap
- Per-thread profiles (treating `RenderThread` vs `GameThread` differently).
- Engine auto-detection (Unity/Unreal/native).
- `debugfs`/tracepoint observability (`trace_hydra_thread_boost`).

## License
GPL-2.0. Copyright (C) 2026 xMikkkaa.
