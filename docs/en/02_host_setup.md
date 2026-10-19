# 2. Capture host setup

[Back to contents](../index.md#english) · [中文](../zh/02_host_setup.md)

The recorder has two layers: `devices/` (device drivers) and `record_ego/` (capture program + web console). There are two host topologies, one per glove type.

## 2.1 Two topologies

### MANUS rig: one Windows host

```text
VIVE Ultimate Tracker ×5 ──dongle──► Windows: VIVE Hub + SteamVR (headless) ──OpenXR──► recorder (same PC)
MANUS gloves ──► Windows: MANUS Core + modified SDKMinimalClient ──UDP:9000──► recorder (same PC)
GoPro ──HDMI──► capture card ──USB──► recorder (same PC, DirectShow)
```

All three streams are timestamped on **one machine and one wall clock** (`time.time()`), so they are aligned as written.

### Wuji rig: Linux capture PC + Windows tracker PC

```text
VIVE Ultimate Tracker ×5 ──► Windows: VIVE Hub + SteamVR + tracker sender ──TCP:8766──► Linux recorder
Wuji gloves ×2 ──USB/SDK──► Linux: wuji_sdk ──► Linux recorder
GoPro ──HDMI──► capture card ──USB──► Linux recorder (V4L2)
```

Video and gloves use the Linux wall clock. Tracker samples are stamped with the Windows wall clock first; the receiver estimates the clock offset between the two machines with NTP-style round trips and converts to the Linux wall clock (re-estimated per session). Measured in-session drift is 4.2 ms median with 0.39 ms fit jitter, which is good enough.

## 2.2 Windows side (trackers)

1. Install **VIVE Hub** and **SteamVR**, pair the five Ultimate Trackers with the dongle, and build the play-area map following VIVE Hub (inside-out tracking needs it).
2. In SteamVR's tracker management, assign each tracker a role: `camera`, `left_wrist`, `right_wrist`, `left_ankle`, `right_ankle`.
   - The role must be in the driver's supported list or the tracker never appears (`ROLES_FULL` / `ROLES_CLASSIC` in `devices/vive_tracker.py`).
   - **Trackers with the role "Handheld object" are not reported.**
3. SteamVR runs headless. The driver `devices/vive_tracker.py` is built on pyopenxr and derives XrTime from the Win32 performance-counter extension.
4. On the Wuji rig, run the tracker sender on Windows and point it at TCP port 8766 on the Linux capture PC (an optional shared secret goes in `--tracker-token` / `EGO_TRACKER_TOKEN`).

> ⚠️ Known issue: on the MANUS rig the trackers are only recorded at ~15 Hz. The OpenXR session is created with `MND_HEADLESS` and submits zero layers per frame, so the runtime throttles the frame loop to a background cadence (90/6 ≈ 15 Hz), and the recording rate equals how often `xr.wait_frame()` returns. The real fix is to take pose sampling out of the XR frame loop and call `xrLocateSpace` from a high-rate thread. The Wuji rig goes through the Windows sender at ~65 Hz and does not have this problem.

## 2.3 Linux side (Wuji capture PC)

1. **Capture card**: find the device and its formats
   ```bash
   sudo apt install v4l-utils
   v4l2-ctl --list-devices
   v4l2-ctl -d /dev/videoN --list-formats-ext
   ```
2. **Encoding**: recordings use the system ffmpeg's libx264 (`--video-codec h264 --crf 20 --h264-preset ultrafast`). The ffmpeg bundled with opencv-python often lacks libx264 and silently falls back to mp4v, so install the system ffmpeg. Video PTS are set from each frame's grab time (`timestamp_mode = capture-vfr`), so encoder speed never changes the timeline.
3. **Wuji SDK**: install `wuji_sdk`. At startup the recorder imports the calibration bundle (default `wuji_calibration/calibration_bundle.zip`), scans and connects every glove and subscribes to `tactile` + `hand_skeleton` (derived zones, joint angles, EMF and point clouds are off by default to keep video capture real-time).
4. **Network**: the Windows sender must reach TCP 8766 on the Linux PC.
5. **Upload**: configure an rclone remote (e.g. `eailab-hdd2:`); the recorder uploads manually or when a storage threshold is hit via `--rclone-remote` (chapter 4).

## 2.4 Windows capture PC for the MANUS rig

1. Install MANUS Core and run the modified `SDKMinimalClient`, which sends raw skeleton frames over UDP to port 9000 on the same PC (`--manus-ip 0.0.0.0 --manus-port 9000`).
2. MANUS UDP frames only carry an incrementing frame counter and no device clock; timestamps are the host wall clock at packet arrival, which is the same clock as video and trackers.
3. The capture card uses DirectShow (`CAP_DSHOW`) on Windows.

## 2.5 Clocks and synchronisation

- Every timestamp is in **microseconds**; the master timeline is the **host wall clock** `t_host_us`.
- Gloves also store the device clock `t_dev_us` (MANUS has none: −1). On the Wuji rig the tracker also stores the original Windows time `t_source_host_us`; `t_host_us − t_source_host_us` is the clock conversion applied to that sample.
- Three streams, three rates: downstream code always resamples by timestamp and **never aligns by index** (chapters 8 and 12).

## 2.6 Pre-capture checklist

- [ ] All five trackers online in SteamVR with the right roles, no dropouts
- [ ] Both gloves online, left/right slot matches what is worn
- [ ] Recorder page shows video at the expected resolution and frame rate
- [ ] Enough free disk (default cap 50 GB; above it the recorder uploads and cleans up)
- [ ] Wuji rig: the page shows the Windows sender as connected
