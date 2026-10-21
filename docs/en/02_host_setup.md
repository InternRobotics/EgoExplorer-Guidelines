# 2. Capture host setup

[Back to contents](../index.md#english) · [中文](../zh/02_host_setup.md)

The capture code is in `capture/` of [egoexplorer-toolkit](https://github.com/InternRobotics/egoexplorer-toolkit). One code base supports both gloves, with entry points `record_ego_wuji.py` and `record_ego_manus.py`.

## 2.1 Two setups

=== "Wuji rig"

    ```mermaid
    flowchart LR
        W[Wuji gloves ×2] -- Ethernet --> L
        G[GoPro + Media Mod] -- HDMI → capture card --> L
        T[VIVE Ultimate Tracker ×5] --> WIN[Windows: SteamVR<br/>tracker_sender_windows.py]
        WIN -- TCP 8766 --> L[Linux capture PC<br/>record_ego_wuji.py]
        L --> F[session folder]
    ```

    The Linux PC records everything: video and gloves are timestamped with the Linux system clock. A Windows PC reads the trackers and sends each pose to Linux together with its own clock. Every 2 seconds the two sides compare clocks; Linux takes the sample with the lowest round-trip time to estimate the offset between the machines and converts tracker times to Linux time. Until the first estimate, and whenever the offset exceeds 5 seconds (treated as an error), it uses the time Linux received the data instead. Measured clock drift within a recording is 4.2 ms (median).

=== "MANUS rig"

    ```mermaid
    flowchart LR
        M[MANUS gloves] --> MC[MANUS Core<br/>SDKMinimalClient_Windows.exe] -- UDP 9000 --> R
        T[VIVE Ultimate Tracker ×5] -- SteamVR (same PC) --> R
        G[GoPro + Media Mod] -- HDMI → capture card --> R[Windows capture PC<br/>record_ego_manus.py]
        R --> F[session folder]
    ```

    One Windows PC records everything, and all three streams share its system clock, so they are aligned as written. MANUS sends 25-node hand skeletons; the capture program drops the four non-thumb metacarpal nodes and saves 21 nodes (the same layout as Wuji and MediaPipe).

## 2.2 SteamVR and trackers (both setups)

1. Install **VIVE Hub** and **SteamVR**, pair the five trackers with the wireless dongle and build the play-area map as VIVE Hub guides you.
2. In SteamVR, make SteamVR the active **OpenXR Runtime**.
3. Assign each tracker its role from 1.3. A tracker with the role Handheld object outputs no pose. The sender prints the serials and roles it finds at startup, which you can check against.

Tracker sample rate:

| Setup | How | Default |
|---|---|---|
| Wuji | The Windows sender is capped (`--send-fps`), and so is writing on Linux (`--tracker-record-fps`). Without a cap, some OpenXR runtimes produce near 1000 duplicate poses per second, which crowds out glove data on Linux | send 90, write 120 |
| MANUS | Tracker poses are queried on the system clock (`--tracker-sample-fps`) instead of following SteamVR's background refresh of about 15 Hz | 90 |

!!! note
    MANUS recordings made before this sampling fix have trackers at about 15 Hz (6.5).

## 2.3 Linux capture PC (Wuji rig)

Ubuntu 22.04 or later (required by the Wuji SDK), Python 3.10 or 3.11.

```bash
cd capture
python3 -m venv .venv && source .venv/bin/activate
python -m pip install -r requirements.txt
```

### Connecting the gloves

Each glove uses its own network port (usually a USB Ethernet adapter). The factory addresses are `192.168.1.100` for the left glove and `192.168.1.101` for the right. The setup script finds the right ports, writes a persistent network configuration and verifies it by scanning and connecting once:

```bash
# configure and verify
./scripts/setup_wuji_gloves.sh
# check only, change nothing
./scripts/setup_wuji_gloves.sh --check
# name the ports if detection is ambiguous
./scripts/setup_wuji_gloves.sh --interface enx001 --interface enx002
# require both gloves; open the SDK ports if UFW is on
./scripts/setup_wuji_gloves.sh --expected-gloves 2 --allow-ufw
```

The script never clears existing addresses and never touches the port that carries the default route. If the glove IPs were changed, pass `--left-ip`, `--right-ip` and `--host-prefix`.

### Glove calibration

Each pair of gloves needs a hand calibration before the SDK has a hand model for these gloves and their wearer. The calibration is stored under a Wuji SDK "user". At startup the capture program imports `wuji_calibration/calibration_bundle.zip` and switches to the user who owns it (`--wuji-calibration` for another path; `--no-wuji-calibration` skips the import and uses the SDK's current user). The bundle is not in the repository; produce it as follows (requires `wuji-sdk>=2026.7.21`):

1. Close Wuji Studio (it holds the SDK) and check with `setup_wuji_gloves.sh --check` that both gloves are found.
2. Create an SDK user for this pair of gloves and note the returned `user_id`:
    ```bash
    python scripts/wuji_calibration_tool.py users                 # existing users, * marks the current one
    python scripts/wuji_calibration_tool.py create-user "rig-A"
    ```
3. Calibrate, in either of two ways:
    - **Wuji Studio (recommended)**: open Wuji Studio, select the user from step 2, calibrate each glove following the [Wuji Studio documentation](https://docs.wuji.tech/docs/en/wuji-studio/latest/), then close Wuji Studio;
    - **Script**: `python scripts/wuji_calibration_tool.py calibrate --user <user_id>`; the terminal tells you each pose to make.
4. Export and check:
    ```bash
    python scripts/wuji_calibration_tool.py export --user <user_id>   # writes wuji_calibration/calibration_bundle.zip
    python scripts/wuji_calibration_tool.py preview                   # check only, no import
    ```
5. Copy `calibration_bundle.zip` into `capture/wuji_calibration/` on the other capture machines. The capture program imports it at startup and prints the loaded calibration user. `wuji_calibration_tool.py import` imports it by hand.

Recalibrate and export again after changing or repairing gloves.

### Capture card

```bash
sudo apt install v4l-utils
v4l2-ctl --list-devices                        # find the card's /dev/videoN
v4l2-ctl -d /dev/videoN --list-formats-ext     # supported resolutions and frame rates
```

Frame grabbing and encoding run in separate threads. Each MP4 frame's display time is its grab time, so dropped frames never make the video play faster.

### Network

The Windows sender must reach TCP port 8766 on Linux:

```bash
hostname -I                                                  # Linux LAN IP
sudo ufw allow from <WINDOWS_IP> to any port 8766 proto tcp  # if UFW is on
```

Never expose port 8766 to the internet. Even on a LAN, set the same `--tracker-token` (or `EGO_TRACKER_TOKEN`) on both sides.

## 2.4 Windows capture PC (MANUS rig)

MANUS glove data passes through two programs before it reaches the capture program:

- **MANUS Core**: MANUS's own glove service. It connects the gloves, runs the hand calibration and computes finger poses. Download it from the [MANUS resources page](https://docs.manus-meta.com/3.0.0/Resources/) and pair and calibrate the gloves as described there.
- **UDP sender** (`SDKMinimalClient_Windows.exe`): built on the MANUS SDK, it reads glove data from MANUS Core and sends it over UDP to `record_ego_manus.py`. It and `ManusSDK.dll` fall under the MANUS SDK licence and are not in the repository; obtain them and put them in `capture/ManusClient/`. The `README.md` in that folder explains how to get them and documents the UDP packet format.

Dependencies are the same `requirements.txt`; on Windows it installs `pyopenxr` and skips `wuji-sdk`. The capture card is read through DirectShow on Windows.

Start order: MANUS Core (gloves online) → UDP sender → capture program.

## 2.5 Pre-capture checklist

=== "Wuji rig"

    - [ ] All five trackers online in SteamVR with the right roles
    - [ ] `setup_wuji_gloves.sh --check` passes, both gloves found, Wuji Studio closed
    - [ ] `wuji_calibration/calibration_bundle.zip` belongs to this pair of gloves
    - [ ] The capture page shows the tracker connection, a camera picture and live tracker poses
    - [ ] Enough free disk

=== "MANUS rig"

    - [ ] All five trackers online in SteamVR with the right roles
    - [ ] Gloves online in MANUS Core; the UDP sender running with the right target IP and port
    - [ ] The capture page shows the tracker connection, a camera picture and live tracker poses
    - [ ] Enough free disk
