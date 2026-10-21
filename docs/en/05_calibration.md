# 5. Calibration

[Back to contents](../index.md#english) · [中文](../zh/05_calibration.md)

## 5.1 Why calibrate separately

A tracker reading depends on two things at once: where and in which orientation the tracker sits on the limb (its mounting), and how the limb itself moves (the joint pose). Fitted together, mounting errors get absorbed as pose errors. So after each donning, a prescribed sequence is used to solve body shape, scale and the mounting of the five trackers on their own, and every later task recording reuses that result.

The result is saved as `subject_profile.json`:

- SMPL-X shape `betas` (10) and `scale`;
- the mounting translation and rotation of each tracker relative to its joint;
- body dimensions, a personal turning and walking reference (from stages 16–17), and per-tracker residuals.

## 5.2 When to recalibrate

- A different wearer;
- any tracker removed and re-attached, or a strap that visibly slipped;
- new gloves or new hand bases.

As long as the mounting is unchanged, one calibration can be reused across days and tasks. A new wearer or a new donning does **not** require retraining the network: the network's input already has the mounting removed.

## 5.3 The calibration sequence

The sequence has 17 stages and takes about 144 seconds. Hold each static pose for at least 3 seconds; move slowly, smoothly and within a comfortable range. The wearer follows the three-view reference video below (`assets/calibration_protocol_reference.mp4` in the toolkit).

<video src="../../assets/video/calibration_protocol_reference.mp4" controls muted loop playsinline width="100%"></video>

![The 01–17 calibration reference](../assets/img/protocol-fig.jpg)

*The reference motion is generated with SMPL-X; the numbers are times within the reference. Left and right are the wearer's own; nothing is mirrored.*

| Stage | Duration | Motion | Solves for |
|---|---:|---|---|
| 01 | 7 s | Stand naturally, toes forward, arms down, looking ahead | Floor, height, initial heading |
| 02 | 9 s | Body still; slowly turn the head left, right, down, up, pausing 1 second at each end | Head mounting rotation |
| 03 | 7 s | A-pose: arms about 45° down and out, palms down | Shoulder width, upper arm |
| 04 | 7 s | T-pose: arms horizontal, elbows straight, palms down | Arm span, shoulder-to-wrist length |
| 05 | 7 s | Arms straight forward, palms down | Hand tracker position and orientation |
| 06 | 8 s | Arms out to the side with palms up; bend the elbows right, left, right, left | Upper and forearm lengths, hand mounting |
| 07 | 8 s | Arms forward; turn the wrists palm down, over, inward, outward | Hand tracker mounting rotation |
| 08 | 8 s | Two shallow squats, torso upright, feet still | Hip, knee and ankle lengths, floor |
| 09 | 7 s | Raise and lower the left leg twice without leaning | Left ankle mounting |
| 10 | 7 s | Raise and lower the right leg twice | Right ankle mounting |
| 11 | 7 s | In place, rise onto the toes, then lift the toes | Ankle tracker mounting rotation |
| 12 | 7 s | Hands forward; spread and close the fingers | Palm scale, glove zero position |
| 13 | 7 s | Slowly make a fist and open fully, twice | Finger flexion scale and zero |
| 14 | 9 s | Touch the thumb to the index, middle, ring and little finger in turn, then open | Opposition and pinch |
| 15 | 9 s | Fully bend and straighten the index, middle, ring and little finger in turn | Per-finger mapping |
| 16 | 16 s | Small steps on the spot: turn 180° left and back, then 180° right and back, head turning with the body | Heading, turning continuity, left/right and front/back check |
| 17 | 14 s | Facing forward, take two steps forward, then two steps back to the start | Gait, pelvis translation, floor contact check |

Recording requirements:

- A flat, hard floor, with tape marking where each foot goes and the toes pointing the same way. The opening stance defines "left" and "forward" in the world frame (11.1).
- After switching on, let the trackers settle and check that all five share one coordinate frame without jumps.
- Stand still for 3 seconds before starting.
- Trackers must not be removed or moved during calibration or during the task recordings that follow.

Stages 16–17 check the full reconstruction and give the personal turning and walking reference; they do not enter the solve for shape and mounting, because the pelvis keeps moving while turning and walking, and treating those frames as static poses would bias the result.

## 5.4 Running the calibration

Run from the toolkit root; a CPU is enough:

```bash
PYTHONPATH=src python -m five_tracker_smplx.cli calibrate-subject \
  --data <calibration recordings of one wearer and one donning> \
  --height <height_m> --armspan <armspan_m> \
  --output-dir $WORK/profiles/<name> \
  --body-model-path models/smplx \
  --vposer-path models/vposer/V02_05 \
  --motion-cache assets/calibration_motion_vposer_mano.npz \
  --iterations 1100 --optimized-betas 5 --device cpu
```

- **Stages are aligned automatically.** Stages 01–11 are recognised from the trackers' positions and rotations, 12–15 also use glove joint motion, and 16–17 use the turning and displacement of the head and ankles. Each recording gets a `protocol_alignment.json` with stage boundaries and confidence. If automatic alignment fails, stage boundaries can be annotated by hand (`*_manual_alignment.json`).
- **Shape prior (optional).** `--smplx-prior <npz>` uses an SMPL-X shape estimated from video (for example by GVHMR) as a prior, with weight 0.18 by default. Without it the mounting barely changes (a few millimetres), but shape and scale differ.
- **How it solves.** Each tracker has one mounting translation and rotation for the whole sequence. Only the first few shape components are free, with height and arm span as strong constraints: all ten components cannot be recovered from five trackers. The tilt of the pelvis-to-neck line relative to gravity is only a soft constraint.

The output folder holds `subject_profile.json` and an acceptance page, `subject_calibration.html`. Once it passes, copy `subject_profile.json` to `$WORK/profiles/<batch>.json`, where chapter 7 picks it up.

## 5.5 Acceptance

Open `subject_calibration.html` and check:

- [ ] stages 01–17 are in order with correct boundaries;
- [ ] turning the head does not turn the torso; elbow, wrist, squat, leg-raise and ankle motions look natural, with no left/right swap;
- [ ] feet flat and still in the static stages; turns made of alternating small steps; the walk is two steps forward and two back, four footfalls in total;
- [ ] no unusual pelvis sway and no jumps between neighbouring frames;
- [ ] each tracker's residual stays steady in the static stages and does not grow over time.

If one tracker's residual stays above 3–4 cm, check first whether its strap slipped and whether its role is right, then the coordinate frame.

## 5.6 Which calibration a recording uses

A calibration belongs to a wearer and a donning, not to a rig or a task. Processing groups recordings (7.3): one group holds recordings of one wearer in one donning, and the whole group uses that donning's calibration.

- The same rig worn by someone else is a new group and needs a new calibration;
- record in the capture log who made each recording and in which donning;
- if the log is missing, look at how far the head tracker sits above the ankle trackers when standing. It differs between people by several centimetres; compare it with each calibration recording.

A wrong calibration costs a lot: one recording went from 62 mm to 80 mm residual with someone else's calibration.
