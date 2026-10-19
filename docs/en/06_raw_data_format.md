# 6. Raw data format

[Back to contents](../../README.md#english) · [中文](../zh/06_raw_data_format.md)

Each session is a folder `session_<date>_<time>_<task>/` with five files:

```text
<session>.mp4            GoPro video (H.264, PTS from grab time, VFR)
<session>.video.npz      per-frame host timestamps and encoder statistics
<session>.tracker.npz    global 6-DoF poses of the five trackers
<session>.npz            both gloves (skeleton; Wuji adds touch)
<session>.session.json   device and clock metadata
```

The cluster copy (`/data/ego_data_sync/`) **has no mp4** (too large; review pages fetch a 360p copy from the bucket when needed).

All timestamps are in **microseconds**; `t_host_us` is the host wall clock (`time.time()`) and is the shared master timeline.

## 6.1 `*.tracker.npz`

The key prefix is always `tracker_0__pose`:

| Key | Shape | Meaning |
|---|---|---|
| `tracker_0__pose__t_host_us` | (T,) int64 | Host wall clock (already converted to the Linux clock on the Wuji rig) |
| `tracker_0__pose__t_us` / `__t_dev_us` | (T,) int64 | Device-preferred time / device time |
| `tracker_0__pose__t_source_host_us` | (T,) int64 | Wuji rig only: original Windows wall clock |
| `tracker_0__pose.<role>.position` | (T, 3) float32 | Metres, tracker world frame (+Y up) |
| `tracker_0__pose.<role>.orientation` | (T, 4) float32 | **Quaternion in `w, x, y, z` order**; may be NaN on invalid frames |
| `tracker_0__pose.<role>.valid` | (T,) bool | Whether the sample is valid |
| `meta` | scalar string | JSON: roles, frames, etc. |

`<role>` ∈ `camera, left_wrist, right_wrist, left_ankle, right_ankle`.

> ⚠️ Raw quaternions are **wxyz**; released LeRobot data and glove skeletons are **xyzw**. Read raw files with the library's `raw_trackers.read_raw_tracker_npz()`, which reorders them. Also, some frames carry zero-norm quaternions while still marked valid, and a few sessions contain NaN; filter on `valid & finite`.

## 6.2 `*.npz` (gloves)

Keys follow `{device}__{stream}__t_us / __t_dev_us / __t_host_us` and `{device}__{stream}`. `glove_0` / `glove_1` are the two hands; **which one is left is given by `slot` in `session.json`** (do not assume glove_0 = left).

| Key | Wuji | MANUS (new, 21-node) | MANUS (old, 25-node) |
|---|---|---|---|
| `glove_*__hand_skeleton` | (T, 21, 7) | (T, 21, 7) | (T, 25, 7) |
| `glove_*__tactile` | (T, 744) | — | — |
| `glove_*__hand_joint_angles.angles` / `.conf` | — | (T, 20) / (T, 5) | present |
| Rate | ~120 Hz | skeleton ~220 Hz, angles ~200 Hz | ~22 Hz |

- Each `hand_skeleton` row is `[px, py, pz, qx, qy, qz, qw]` (position + **xyzw** quaternion); node 0 is the wrist.
- **The 21-node layout is MediaPipe**, parents `[-1,0,1,2,3,0,5,6,7,0,9,10,11,0,13,14,15,0,17,18,19]`; the 25-node MANUS layout adds four non-thumb metacarpal nodes. Loaders decide the layout **by node count**, not by device type (new MANUS exports are also 21-node).
- Wuji node 0 always has an identity quaternion and **carries no global hand orientation**; hand orientation can only come from the wrist tracker. MANUS node 0 does carry a world orientation.
- Wuji touch: 744 = a 24×31 matrix flattened row-major; `-1` marks invalid (unwired) taxels, about 28%. Raw touch values have a non-zero baseline.
- `meta`: JSON with `device_type` (`wuji_glove` / `manus_glove`), each hand's `sn` and `slot`, per-stream frame counts and durations, `host_minus_dev_us`.

## 6.3 `*.video.npz`

| Key | Meaning |
|---|---|
| `t_host_us` | (N,) host wall clock at each frame grab |
| `fps` / `width` / `height` | **Nominal** values; compute the real rate from `t_host_us` (Wuji rig: nominal 60, measured ~30) |
| `timestamp_mode` | `capture-vfr` |
| `captured_frames`, `queue_dropped_frames`, `read_failures`, `writer_failures` | Capture statistics; all but the frame count should be 0 |

## 6.4 `*.session.json`

Key fields: `base`, `task_name`, `glove_type` (`wuji`/`manus`), `clock` (clock description), `video{…}`, `gloves[{device_name, sn, slot}]`, `tracker{roles, transport, source_clock, host_clock, dropped_frames, record_fps_limit}`, each stream's `t_host_us_range`, and `overlap_s` (how long all streams overlap).

Left/right correction (chapter 9) only rewrites this file in the corrected copy (swapping `slot`) and records the evidence; originals are never touched.

## 6.5 Measured rates (know these before writing code)

| Rig | Video | Gloves | Trackers |
|---|---|---|---|
| MANUS (older batches) | ~52–60 Hz | ~22 Hz (25-node) / ~220 Hz (21-node) | **~15 Hz** |
| Wuji | **~30 Hz** (nominal 60) | ~120 Hz | ~65 Hz |

Three rates in one recording, and a 4× tracker difference across rigs. Resample by timestamp at every stage (`np.interp` / Slerp); **never align by index and never assume a fixed rate**.
