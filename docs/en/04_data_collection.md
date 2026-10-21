# 4. Capture workflow

[Back to contents](../index.md#english) · [中文](../zh/04_data_collection.md)

<video src="../../assets/video/tasks.mp4" controls muted loop playsinline width="100%"></video>

*Ten tasks, one rig, one calibration protocol. Top row: six laboratory procedures; bottom row: four household and industrial tasks.*

## 4.1 A capture day

1. **Wear the rig and run the checks** (1.4 and 2.5).
2. **Record calibration.** Every donning starts with the 01–17 calibration sequence (chapter 5); one or two spare takes are a good idea. Calibration recordings must be recognisable by name, for example with the task name "calibration".
3. **Record the task list** (chapter 3). Pick the task on the page, `r` to start, `e` to save; `d` discards a bad take.
4. **Spot-check.** Replay a few recordings every day with `visualize_session.py` (4.4).
5. **Upload** (optional, 3.6).
6. **Keep a log.** The capture program writes `recording_log_<date>.csv` next to the data folder with each recording's start and end time, duration and frame counts. Also note by hand who made each recording and in which donning: processing groups recordings by exactly that, one calibration per group (7.3).

## 4.2 Naming

Each recording is a folder named `session_<date>_<time>_<task>`, for example `session_20260901_154755_物料称量`. It holds five files, described in chapter 6:

```text
data/session_20260901_154755_物料称量/
├── session_20260901_154755_物料称量.mp4
├── session_20260901_154755_物料称量.video.npz
├── session_20260901_154755_物料称量.npz
├── session_20260901_154755_物料称量.tracker.npz
└── session_20260901_154755_物料称量.session.json
```

Recordings that deliberately capture interruptions and recovery get `_恢复` ("recovery") after the task name, for example `session_20260901_164121_过柱_恢复`.

## 4.3 Recording rules

- **Stand still for 2–3 seconds** before starting and hold 1–2 seconds at the end. The side checks and the projection both use the opening stance.
- **Do not touch trackers or straps while recording.** If one gets knocked, note it in the log and recalibrate if needed.
- **Make sure the wrist trackers move.** A recording without wrist tracker data cannot be reconstructed.
- **Keep recordings a manageable length.** Laboratory tasks usually take 2–7 minutes; split longer ones.
- **Mind the top of the frame.** The GoPro's HDMI output carries its on-screen status (battery, SD card and so on) along the top, and the release crops the top 110 pixels. Keep important content away from the very top.

## 4.4 Checking a recording

Start with `session.json`: `video.queue_dropped_frames`, `read_failures`, `writer_failures` and `tracker.dropped_frames` should all be 0, and `overlap_s` should be close to the recording length.

Then replay it:

```bash
python visualize_session.py data/session_YYYYMMDD_HHMMSS_<task>
# optional: --no-open --port 8000 --max-fps 30 --build-only
```

The replay page detects the glove type and puts video, the five trackers, both hand skeletons and touch on one timeline (the page is shown in 3.4). What to look for:

| Check | Normal |
|---|---|
| Trackers | All five have coordinates and move, and turn together when the wearer turns |
| Hand skeletons | Both move; the hand moving in the video moves on its own side |
| Touch (Wuji) | Lights up during grasps; large areas that stay on or stay dark point to a glove or calibration problem |
| Duration and rates | Match the recording |
