# 6. Raw data format

[Back to contents](../index.md#english) · [中文](../zh/06_raw_data_format.md)

Each recording is a folder `session_<date>_<time>_<task>/` with five files:

| File | Contents |
|---|---|
| `<session>.mp4` | GoPro video, H.264, variable frame rate; each frame's display time is its grab time |
| `<session>.video.npz` | Grab time of each frame and encoder statistics |
| `<session>.tracker.npz` | Global poses of the five trackers |
| `<session>.npz` | Both glove skeletons; touch on Wuji |
| `<session>.session.json` | Device and clock information |

Reconstruction only reads the tracker, glove and video-timestamp files; the mp4 is needed only for review pages and the LeRobot conversion.

All timestamps are in **microseconds**. `t_host_us` is the capture host's system clock (`time.time()`), and all streams are aligned on it.

## 6.1 Trackers: `*.tracker.npz`

Keys start with `tracker_0__pose`; `<role>` is one of the five roles in 1.3.

| Key | Shape | Meaning |
|---|---|---|
| `tracker_0__pose__t_host_us` | (T,) int64 | Capture host time (converted to the Linux clock on the Wuji rig) |
| `tracker_0__pose__t_us`, `__t_dev_us` | (T,) int64 | Device-preferred timestamp, device time (trackers have no device clock: −1) |
| `tracker_0__pose__t_source_host_us` | (T,) int64 | Wuji rig only: original Windows time |
| `tracker_0__pose.<role>.position` | (T, 3) float32 | Position in metres, world frame, +Y up |
| `tracker_0__pose.<role>.orientation` | (T, 4) float32 | Quaternion, **order w, x, y, z**; may be NaN on invalid frames |
| `tracker_0__pose.<role>.valid` | (T,) bool | Whether the sample is valid |
| `meta` | string | JSON with roles, frame count, etc. |

!!! warning "Quaternion order"
    Raw tracker files are **wxyz**; glove skeletons, calibration files and the released LeRobot data are **xyzw**. Read raw files with the library's `raw_trackers.read_raw_tracker_npz()`, which converts the order. Some frames also have zero-length quaternions while marked valid, and a few recordings contain NaN; use only samples that are valid and finite.

## 6.2 Gloves: `*.npz`

Keys follow `{device}__{stream}__t_us`, `__t_dev_us`, `__t_host_us` and `{device}__{stream}`. `glove_0` and `glove_1` are the two gloves; **which one is the left hand is given by `slot` in `session.json`**, not by the name.

| Key | Wuji | MANUS (current, 21 nodes) | MANUS (older, 25 nodes) |
|---|---|---|---|
| `glove_*__hand_skeleton` | (T, 21, 7) | (T, 21, 7), reduced from 25 nodes at capture | (T, 25, 7) |
| `glove_*__tactile` | (T, 744) | — | — |
| `glove_*__hand_joint_angles.angles`, `.conf` | — | (T, 20), (T, 5) | present |
| Rate | about 120 Hz | skeleton about 220 Hz, angles about 200 Hz | about 22 Hz |

- Each `hand_skeleton` row is `[px, py, pz, qx, qy, qz, qw]`: position plus an **xyzw** quaternion. Node 0 is the wrist.
- The 21-node layout is MediaPipe's, with parents `[-1,0,1,2,3,0,5,6,7,0,9,10,11,0,13,14,15,0,17,18,19]`. The older 25-node layout adds four non-thumb metacarpal nodes. Decide the layout from the node count, not the device type.
- On Wuji gloves node 0 always has an identity rotation and carries no overall hand orientation, so the hand's orientation can only come from the wrist tracker. MANUS node 0 does carry orientation.
- Wuji touch: 744 = 24 × 31, flattened row by row; `-1` marks an invalid taxel, about 28% of them. Raw touch values have a baseline and are not zero without contact.
- `meta` is JSON with `device_type` (`wuji_glove` or `manus_glove`), each glove's serial `sn` and `slot`, and per-stream frame counts and durations.

## 6.3 Video timestamps: `*.video.npz`

| Key | Meaning |
|---|---|
| `t_host_us` | (N,) grab time of each frame |
| `fps`, `width`, `height` | Nominal values. Compute the real frame rate from `t_host_us`: nominal 60 on the Wuji rig, about 30 measured |
| `timestamp_mode` | `capture-vfr` |
| `captured_frames`, `queue_dropped_frames`, `read_failures`, `writer_failures` | Capture statistics; all but the frame count should be 0 |

## 6.4 Device information: `*.session.json`

Main fields: `base` (recording name), `task_name`, `glove_type` (`wuji` or `manus`), `clock` (clock description), `video`, `gloves` (each glove's `device_name`, `sn`, `slot`), `tracker` (roles, transport, source and host clocks, dropped frames, write cap), each stream's `t_host_us_range`, and `overlap_s` (how long all streams overlap).

Left/right correction (9.2) only rewrites this file in the corrected copy; the original stays untouched.

## 6.5 Measured rates

| Rig | Video | Gloves | Trackers |
|---|---|---|---|
| MANUS (early recordings) | 52–60 Hz | about 22 Hz (25 nodes) / about 220 Hz (21 nodes) | **about 15 Hz** |
| MANUS (current capture program) | about 60 Hz | about 220 Hz | 90 Hz by default |
| Wuji | **about 30 Hz** (nominal 60) | about 120 Hz | about 65 Hz |

Streams within one recording run at different rates, and tracker rates differ by more than 4× between rigs. Always resample by timestamp (linear interpolation for positions, Slerp for rotations); never align by array index and never assume a fixed frame rate.
