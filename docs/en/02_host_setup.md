# 2. Capture host setup

[Back to contents](../index.md#english) · [中文](../zh/02_host_setup.md)

The capture code is in `capture/` of [egoexplorer-toolkit](https://github.com/InternRobotics/egoexplorer-toolkit). One code base supports both **Wuji** and **MANUS** gloves, with entry points `record_ego_wuji.py` and `record_ego_manus.py`.

## 2.1 Two topologies

### Wuji rig: Linux capture PC + Windows tracker PC

```mermaid
flowchart LR
    W[Wuji gloves ×2] -- Ethernet / USB NICs --> L
    G[GoPro + Media Mod] -- HDMI → capture card --> L
    T[VIVE Ultimate Tracker ×5] --> WIN[Windows: SteamVR / OpenXR<br/>tracker_sender_windows.py]
    WIN -- TCP 8766 · poses + Windows wall clock --> L[Linux capture PC<br/>record_ego_wuji.py]
    L --> F[session folder<br/>mp4 / video.npz / npz / tracker.npz / session.json]
```

- Linux is the only capture coordinator. Video and gloves are stamped with the Linux wall clock directly.
- Windows sends every pose with its own wall-clock time. Every 2 s the two sides take an NTP-style four-timestamp sample; Linux estimates the `Windows − Linux` offset from the lowest-RTT sample and converts tracker times to the Linux clock. Before the first estimate, and if an offset above 5 s is detected, it falls back to Linux receive time.
- Measured in-session drift is 4.2 ms median with 0.39 ms fit jitter.

### MANUS rig: one Windows PC

```mermaid
flowchart LR
    M[MANUS gloves] --> MC[MANUS Core + SDKMinimalClient_Windows.exe] -- UDP 9000 --> R
    T[VIVE Ultimate Tracker ×5] -- SteamVR / OpenXR (same PC) --> R
    G[GoPro + Media Mod] -- HDMI → capture card --> R[record_ego_manus.py]
    R --> F[session folder]
```

All three streams are stamped on one machine and one wall clock, so they are aligned as written. The MANUS sender emits 25-node skeletons; the recorder drops the four non-thumb metacarpals and writes **21 nodes (the Wuji / MediaPipe layout)**.

## 2.2 Windows side: SteamVR and trackers

1. Install **VIVE Hub** and **SteamVR**, pair the five Ultimate Trackers with the dongle and build the play-area map following VIVE Hub (inside-out tracking needs it).
2. In SteamVR, **make SteamVR the active OpenXR Runtime**.
3. Assign each tracker a role: `camera`, `left_wrist`, `right_wrist`, `left_ankle`, `right_ankle`.
   - **A tracker with the role "Handheld object" yields no usable pose**; use the roles above.
   - The sender prints the serials and roles it enumerates at startup, which you can check against.
4. Install the Windows dependencies (just `numpy`, `msgpack`, `pyopenxr`):
   ```powershell
   python -m venv .venv-tracker
   .\.venv-tracker\Scripts\Activate.ps1
   python -m pip install -r requirements-tracker-windows.txt
   ```

**Tracker rate**:
- Wuji rig: the sender sends at most 90 FPS by default (`--send-fps 90`) and Linux writes at most 120 FPS (`--tracker-record-fps`). The cap matters: some headless OpenXR runtimes return from `wait_frame` immediately, and without a cap they produce near-1000 FPS duplicate poses that starve glove reception on Linux.
- MANUS rig: the local trackers are sampled on the **host clock** (`--tracker-sample-fps`, default 90). The headless SteamVR compositor only runs at about 15 Hz, but `xrLocateSpace` can be queried at any time, so sampling is no longer tied to the frame cadence; a full wait/begin/end frame is only pumped every 0.5 s to keep the session alive. **Older MANUS recordings made before this fix have trackers at about 15 Hz** (6.5).

## 2.3 Linux side (Wuji rig)

Ubuntu 22.04 or newer (required by the Wuji SDK), Python 3.10 / 3.11.

```bash
cd capture
python3 -m venv .venv && source .venv/bin/activate
python -m pip install -r requirements.txt      # numpy / opencv-python / msgpack / av / Pillow / wuji-sdk
```

**Connect the Wuji gloves.** Each glove uses its own Ethernet port (usually a USB Ethernet adapter); the factory addresses are `192.168.1.100` (left) and `192.168.1.101` (right). The setup script configures the NICs and routes and verifies the connection:

```bash
./scripts/setup_wuji_gloves.sh                     # detect ports, create persistent NetworkManager profiles, scan and connect
./scripts/setup_wuji_gloves.sh --check             # check only, change nothing
./scripts/setup_wuji_gloves.sh --interface enx001 --interface enx002   # name the NICs if auto-detection is ambiguous
./scripts/setup_wuji_gloves.sh --expected-gloves 2 --allow-ufw         # require both gloves; open SDK ports if UFW is on
```

The script never flushes existing addresses and never touches the NIC carrying the default route. If the glove IPs were changed, pass `--left-ip`, `--right-ip` and `--host-prefix`.

**Wuji calibration bundle**: at startup the recorder imports the glove calibration from `wuji_calibration/` (`--wuji-calibration` to point elsewhere, `--no-wuji-calibration` to use the SDK's current user). This folder comes from Wuji's own calibration tool and is not in the repository; get it from the person in charge of the devices.

**Capture card**:

```bash
sudo apt install v4l-utils
v4l2-ctl --list-devices
v4l2-ctl -d /dev/videoN --list-formats-ext
```

Frame grabbing and encoding run in separate threads joined by a bounded queue; MP4 PTS come from each frame's grab time `t_host_us` (variable frame rate), so dropped frames never compress the playback timeline.

**Firewall**: the Windows sender must reach TCP 8766 on Linux.

```bash
hostname -I                                               # Linux LAN IP
sudo ufw allow from <WINDOWS_IP> to any port 8766 proto tcp
```

Never expose 8766 to the internet; even on a trusted LAN, set the same `--tracker-token` (or `EGO_TRACKER_TOKEN`) on both sides.

## 2.4 Windows capture PC for the MANUS rig

- Same `requirements.txt` (on Windows it installs `pyopenxr` and skips `wuji-sdk`).
- Run `ManusClient/SDKMinimalClient_Windows.exe` (with `ManusSDK.dll` in the same folder) and point it at the recorder's IP and port (default `0.0.0.0:9000`).
- The capture card uses DirectShow on Windows.

## 2.5 Pre-capture checklist

- [ ] All five trackers online in SteamVR with the right roles
- [ ] Wuji: `setup_wuji_gloves.sh --check` passes and both gloves scan; Wuji Studio is closed (it holds the SDK)
- [ ] MANUS: the UDP sender is running with the right target IP / port
- [ ] The recorder page shows "Tracker connected", with a camera preview and live tracker poses
- [ ] Enough free disk
