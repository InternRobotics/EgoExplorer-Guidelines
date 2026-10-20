# 3. Capture scripts

[Back to contents](../index.md#english) · [中文](../zh/03_capture_scripts.md)

## 3.1 Code layout

```text
capture/
├── record_ego_wuji.py              Wuji rig entry point (Linux)
├── record_ego_manus.py             MANUS rig entry point (Windows)
├── record_ego_app.py               devices, web server and recording orchestration
├── record_ego_page.html            live monitoring and control page
├── tracker_sender_windows.py       Windows tracker sender (Wuji rig)
├── visualize_session.py            replay a recorded session
├── record_ego/                     capture program: config, recording control, session writing, upload
├── devices/                        drivers: wuji_glove / manus_glove / vive_tracker / network_tracker / gopro_camera
├── network_bridge/protocol.py      TCP / MessagePack framing
├── ManusClient/                    MANUS UDP sender (Windows; download it yourself, see README)
├── wuji_calibration/               import / export of Wuji calibration bundles (the bundle is not in the repo)
├── scripts/
│   ├── setup_wuji_gloves.sh        Wuji glove NIC setup and checks
│   ├── wuji_calibration_tool.py    Wuji glove calibration, bundle export / import (2.1.3)
│   ├── start_tracker_sender.ps1    Windows tracker sender launcher
│   ├── wuji_launch.sh              one-command Wuji launch (edit paths for your machine)
│   ├── rclone_linux.sh             upload data (Linux)
│   └── rclone_win.bat              upload data (Windows)
└── task_setting/example.json       task list example
```

## 3.2 Task list

`--task-setting` points to a JSON file holding a non-empty array of task names. The page steps through them; the task name goes into the session name and `session.json`:

```json
["物料称量", "萃取", "开反应", "旋蒸", "烘箱取料", "过柱"]
```

## 3.3 Launching the Wuji rig

**Start the Linux recorder first:**

```bash
cd capture && source .venv/bin/activate
python record_ego_wuji.py \
  --task-setting task_setting/example.json \
  --tracker-host 0.0.0.0 --tracker-port 8766 \
  --save-dir ./data
```

Or edit the paths in `scripts/wuji_launch.sh` and launch with one command.

**Then start the Windows tracker sender** (from `capture/`):

```powershell
.\scripts\start_tracker_sender.ps1 -LinuxHost <LINUX_IP> -Port 8766
# or
python tracker_sender_windows.py --host <LINUX_IP> --port 8766
```

- `host / port` must match on both sides. If the link drops, Windows reconnects every 2 s; the Linux recorder keeps running and video and gloves keep recording.
- Windows only needs outbound LAN access for Python; no inbound port is required.
- **The Windows terminal can also control recording** (over the same authenticated TCP link): `r` starts (with the Linux `--record-countdown`, override with `-Countdown 3` / `--countdown 3`), `e` stops and saves, `Esc` only turns off the shortcut listener while poses keep flowing. Use `-NoKeyboard` / `--no-keyboard` to disable it.

## 3.4 Launching the MANUS rig

```bash
python record_ego_manus.py \
  --task-setting task_setting/example.json \
  --manus-ip 0.0.0.0 --manus-port 9000
```

Trackers are driven locally through `pyopenxr`; without trackers pass `--no-tracker` and the camera and gloves still record.

## 3.5 Control page

<div style="display:flex; gap:12px; align-items:flex-start" markdown>

<div style="flex:1.985 1 0; min-width:0" markdown>

![Capture screen of the MANUS rig](../assets/img/interface.png)

*Capture screen of the MANUS rig*

</div>

<div style="flex:1.778 1 0; min-width:0" markdown>

![Capture screen of the Wuji rig](../assets/img/capture_session_viewer.jpg)

*Capture screen of the Wuji rig*

</div>

</div>

*Both rigs share the same page layout: the GoPro view on the left; the 3D poses and coordinates of the five trackers at top right; the left and right glove skeletons at bottom right (plus the 24×31 tactile pressure heatmaps on the Wuji rig); the control bar at the bottom.*

Default `http://127.0.0.1:8770/` (to open it from another computer, add `--host 0.0.0.0` and open the port). The page shows the camera preview, live tracker poses, glove status, the task switcher and recording buttons; the keys are the same as in the terminal:

| Key | Action |
|---|---|
| `r` | Start a new session after the countdown (default 3 s, adjustable on the page) |
| `e` | Stop and save the current session |
| `d` | During the countdown: cancel; while recording: **stop and discard**, nothing written |
| `q` | Quit; an unsaved recording is discarded |

The Wuji page always shows the camera preview, limited to 640×360 (`--preview-width / --preview-height`); the recording resolution is unaffected.

## 3.6 Common options

| Option | Default | Meaning |
|---|---|---|
| `--cam-device` | `0` | Capture-card index or `/dev/video*` path |
| `--cam-width` / `--cam-height` / `--cam-fps` | `2560` / `1440` / `60` | Requested resolution and rate; the card's actual output is recorded in `session.json` |
| `--video-codec` / `--crf` / `--h264-preset` / `--h264-threads` | `h264` / `20` / `ultrafast` / `4` | Real-time software H.264 |
| `--no-camera` | | No camera |
| `--wuji-calibration` / `--no-wuji-calibration` | default bundle | Wuji glove calibration |
| `--tracker-host` / `--tracker-port` / `--tracker-token` | `0.0.0.0` / `8766` / empty | Tracker receiver (Wuji rig) |
| `--tracker-record-fps` | `120` | Linux-side tracker write cap |
| `--tracker-sample-fps` | `90` | Local tracker sampling rate (MANUS rig) |
| `--no-tracker` | | No trackers |
| `--save-dir` | `./data` | Session root folder |
| `--record-countdown` | `3` | Countdown before recording (s) |
| `--rclone-remote` / `--rclone-config` | empty | Upload target (optional) |
| `--storage-limit-gb` / `--storage-check-interval` | `50` / `30` min | Above the cap, upload and remove verified files |
| `--host` / `--web-port` | `127.0.0.1` / `8770` | Control page address and port |

Full option lists: `python record_ego_wuji.py --help` / `python record_ego_manus.py --help`.

## 3.7 Upload (optional)

Data stays on the local machine under `--save-dir` by default; **uploading is not required**. If local storage is not enough, set up your own cloud object storage (Alibaba Cloud OSS, Tencent Cloud COS, AWS S3 or any S3-compatible service), configure rclone locally and upload to it:

1. Install [rclone](https://rclone.org/downloads/).
2. Run `rclone config` and follow the wizard to create a remote with the storage type, endpoint and AccessKey / SecretKey, e.g. named `myremote`.
3. Verify: if `rclone lsd myremote:<bucket>` lists the contents, the setup works.

Once configured, there are two ways to upload:

- **From the recorder**: start it with `--rclone-remote myremote:<bucket>/ego_data` (or set `EGO_RCLONE_REMOTE`) and the "Upload data" button on the control page becomes available; it runs `rclone copy`, keeping local files and skipping files already on the remote. In addition, when the data folder exceeds `--storage-limit-gb` (default 50 GB, `0` disables it) the recorder runs `rclone move` automatically, deleting only local files confirmed on the remote. Use `--rclone-bin` if rclone is not on PATH and `--rclone-config` if its config file is not in the default location.
- **With the scripts**: `scripts/rclone_linux.sh` / `scripts/rclone_win.bat` are thin wrappers around `rclone copy --progress`: set `LOCAL_PATH` (local data folder) and `REMOTE_PATH` (remote path) at the top and run them. For filters or delete-after-upload, edit the one rclone command (`--include`, `move`).

Without rclone configured, the "Upload data" button is greyed out and nothing is cleaned up automatically when storage exceeds the cap, so keep an eye on disk space yourself.

## 3.8 Troubleshooting

| Symptom | Check first |
|---|---|
| Linux keeps showing "waiting for tracker" | Windows `--host` is the Linux LAN IP (not 127.0.0.1); ports match; no token set on Linux (or the same token on both sides); `Test-NetConnection <LINUX_IP> -Port 8766`; Linux firewall; both machines on one subnet; Linux was started first |
| No tracker poses on Windows | Dependencies installed; SteamVR is the active OpenXR runtime; trackers paired with roles assigned (not Handheld object) |
| Linux cannot find the Wuji gloves | `wuji-sdk` installed and nothing else holding the SDK; `setup_wuji_gloves.sh --check`; glove serials and the current SDK user's calibration |
