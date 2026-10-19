# 1. Hardware and wearing

[Back to contents](../index.md#english) · [中文](../zh/01_hardware_and_wearing.md)

## 1.1 Bill of materials

| Part | Model / spec | Qty | Notes |
|---|---|---|---|
| Body trackers | VIVE Ultimate Tracker | 5 | Inside-out 6-DoF using onboard cameras, **no base stations**; wireless dongle on the Windows host |
| Head camera | GoPro HERO13 Black + Media Mod | 1 | The Media Mod provides the micro-HDMI output used for live capture |
| HDMI capture card | USB UVC capture card | 1 | GoPro → micro-HDMI → HDMI cable → capture card → capture host (`/dev/videoN` on Linux) |
| Gloves (one of) | MANUS Metagloves Pro | 1 pair | Finger articulation only, no touch |
| | Wuji gloves | 1 pair | Finger articulation (21-node MediaPipe layout) + a 24×31 tactile matrix per hand |
| Head mount | 3D-printed adapter + head strap | 1 | Couples the GoPro and the head tracker **rigidly** |
| Hand mounts | 3D-printed base + strap | 2 | Tracker on the glove dorsum |
| Ankle mounts | 3D-printed base + strap | 2 | Tracker on the **lateral** side of the ankle |
| Capture host(s) | see chapter 2 | 1–2 | Wuji rig: Linux capture PC + Windows tracker PC; MANUS rig: one Windows PC |

> STL files and exact dimensions of the printed parts: **to be added** (released with the toolkit).

## 1.2 Tracker roles

Role names are fixed. The recorder, the pipeline and the release format all use the same names, and **no stage may swap them according to the left/right of an image**:

| Recorder role | Pipeline role | Where it is worn | SMPL-X anchor joint |
|---|---|---|---|
| `camera` | `head` | Head, rigidly coupled to the GoPro | 15 (head) |
| `left_wrist` | `left_hand` | Back of the **subject's own** left hand | 20 (left_wrist) |
| `right_wrist` | `right_hand` | Back of the subject's own right hand | 21 (right_wrist) |
| `left_ankle` | `left_foot` | Subject's own left lateral ankle | 7 (left_ankle) |
| `right_ankle` | `right_foot` | Subject's own right lateral ankle | 8 (right_ankle) |

"Left" always means the subject's own left. When you face the subject, their left hand is on your right; that is ordinary perspective, not a swap.

## 1.3 Why the camera is rigid to the head tracker

The head tracker measures the camera's 6-DoF pose directly, so the camera pose is a **measurement** rather than something solved later with visual SLAM. This only holds if the two never move relative to each other: tighten the adapter and keep the head strap snug. If the camera is knocked during a recording, its extrinsics are no longer trustworthy for that part.

## 1.4 Putting the rig on

Two people work best: one wears, one checks.

1. **Measure the subject** (only for a new subject or a new calibration round; cm, each measured twice and averaged): height, arm span (middle fingertip to middle fingertip), biacromial width, neck base to crotch, crotch to floor, left/right upper arm, left/right forearm, left/right hand length (wrist crease to middle fingertip), palm width, left/right ankle-tracker centre to floor. Record left and right separately if they differ noticeably. Calibration itself uses **height and arm span** (entered in metres); the rest are for reference.
2. **Put on the gloves and check left/right.** Wuji gloves are told apart by serial number (e.g. `WG1JA…` left, `WG1KA…` right; check against the actual devices), and the `slot` in the recorder's `session.json` must match.
3. **Mount the hand trackers** at the centre of the glove dorsum, strapped so they do not wobble when the wrist moves. **The left hand must carry the tracker whose SteamVR/VIVE role is `left_wrist`**, and likewise on the right.
4. **Mount the ankle trackers** on the lateral side above the ankle bone, strapped so they neither rotate nor slide when walking or squatting. Watch the roles.
5. **Put on the head strap + head tracker + GoPro**, adjusted so it does not move; the camera points along a natural forward gaze.
6. **Take two mounting photos** (front and side) showing where each tracker sits and how it is oriented; they help later debugging.
7. **Static self-check**: the subject stands naturally for a few seconds. On the recorder page, confirm that all five trackers are online without dropouts, both gloves stream, and the video shows a picture.

## 1.5 Wearing rules (these decide whether the data is usable)

- **Calibration and task recordings must come from the same donning.** After a tracker is removed and re-attached, a strap slips, or the subject changes, record a new calibration (chapter 5). No coordinate fix can compensate for an unknown mounting rotation after remounting.
- **Do not adjust trackers between tasks.** The mounting extrinsics (tracker translation and rotation relative to the bone) are solved once per donning.
- **Get left and right right.** Past batches contained 17 sessions with swapped trackers and 70 with swapped gloves. Chapter 9 detects and corrects these automatically, but correction is not free; avoid it at the source.
- **A loose strap is worse than none.** A tracker that slowly slides shows up as drift: every frame looks plausible and the optimiser quietly absorbs the error into body pose.
- **Wuji touch** degrades slowly with use, and some lines become intermittent. Start every recording with about 15 s of tactile calibration motion (both hands make a fist, open, press) so a per-session health estimate is possible.
