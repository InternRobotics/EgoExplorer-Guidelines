# 4. Collecting data

[Back to contents](../../README.md#english) · [中文](../zh/04_data_collection.md)

Launch commands are in chapter 3 (to be added); this chapter covers the flow and the rules of a capture day.

## 4.1 A capture day

```text
Wear (ch. 1) → host checks (2.6) → record the 01–17 calibration (ch. 5) → record the task list → upload → fill in the capture log
```

1. **Wear the rig and run the checks.**
2. **Record calibration first**, for every donning (two or three takes in the same donning are a good spare). Calibration and task data go into the same day's folder, but calibration sessions must be recognisable by name.
3. **Record the task list.** The recorder reads the task flow from `--task-setting <json>`, a non-empty array of task names, for example:
   ```json
   ["物料称量", "萃取", "开反应", "旋蒸", "烘箱取料", "过柱"]
   ```
   Pick the current task on the page before recording; the task name goes into the session name and the `task_name` field of `session.json`.
4. **Recording controls** (web buttons and terminal keys are equivalent):

   | Key | Action |
   |---|---|
   | `r` | Start: a countdown first (default 3 s, adjustable on the page), with a beep when it ends |
   | `d` | During the countdown: cancel; while recording: **stop and discard**, nothing is written |
   | `e` | Stop and save: writes the mp4, glove npz, tracker npz, video timestamps and session.json, and appends to the day's recording log |
   | `q` | Quit |

5. **Upload**: after saving, an rclone upload can be triggered by hand (`copy --ignore-existing`, safe to repeat). When the local data folder exceeds the storage cap (default 50 GB) the recorder runs `move --checksum` and only removes files that were verified as uploaded.
6. **Capture log**: the recorder writes `recording_log_<date>.csv` next to the data folder (start/end time, duration and per-stream frame counts per session), used to tally daily hours.

## 4.2 Naming and layout

- Session name: `session_<YYYYMMDD>_<HHMMSS>_<task>`, e.g. `session_20260901_154755_物料称量`.
- Recovery recordings (responses to interruptions or failed attempts) append `_恢复` to the task name, e.g. `session_20260901_164121_过柱_恢复`. They are recorded deliberately so the data covers how people correct course.
- One folder per session with five files (format in chapter 6).
- The bucket has one folder per dataset: `eailab-hdd2:/eailab/ego_data/<dataset>/…` (e.g. `wuji`, `manus_new`, `manus_L1`). **Folder depth differs between datasets**; always scan recursively downstream.

## 4.3 Recording rules

- **Stand still for 2–3 s at the start** before moving, and hold for 1–2 s at the end. Projection and the left/right check both use the starting stance.
- **Do not touch trackers or straps during a recording.** If one gets knocked, note it in the capture log and re-record calibration if needed.
- **The wrist trackers must be recording.** Some older `_恢复` recordings only had usable video, with wrist trackers that never moved; the body cannot be rebuilt from those.
- **Length**: laboratory tasks typically run 2–7 minutes. Split very long recordings.
- **Keep subjects distinguishable.** The pipeline picks a calibration profile per subject (5.6). If the same rig is worn by someone else, write it in the capture log; otherwise it can only be inferred afterwards from head height.

## 4.4 After recording

- In `session.json`: `video.queue_dropped_frames`, `read_failures` and `writer_failures` should be 0; `tracker.dropped_frames` should be 0; `overlap_s` should be close to the recording length.
- In the glove `meta`, each stream's `frames` and `duration_s` should match the video length.
- If possible, replay one session on the capture PC to confirm all five trackers and both gloves move.
