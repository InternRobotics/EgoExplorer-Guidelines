# 11. Coordinate conventions

[Back to contents](../index.md#english) · [中文](../zh/11_coordinate_systems.md)

This is the project's single coordinate contract; the code is `src/five_tracker_smplx/coordinates.py`. Three systematic mistakes were made here early on (11.9), so read this chapter in full before reading or writing data.

## 11.1 World frame

| Item | Convention |
|---|---|
| Source | The VIVE / SDK world frame as recorded, with **no origin, yaw or axis transform applied** (`coordinate_frame = raw_capture_y_up`; `sdk-world-as-recorded` in LeRobot) |
| Handedness / units | Right-handed, metres |
| Vertical | **+Y up**, gravity `(0, −1, 0)` |
| Origin | The reference location designated in the VIVE play-area map (neither the floor nor the subject) |
| Floor | At height `ground_y_m` (a Y value in raw coordinates, estimated by default from a low percentile of the ankle trackers) |
| Horizontal heading | During calibration, floor tape aligns the starting stance so that **+X = subject left, +Z = subject forward**. A task recording may start at any yaw; the body's left/forward then comes from the fitted root rotation, and the fixed world X/Z must **not** be read as the body's current left/forward |

## 11.2 Tracker poses

- **Position**: the tracker origin in the world frame (metres).
- **Quaternion order**:

  | File | Order |
  |---|---|
  | Raw `*.tracker.npz` | **w, x, y, z** |
  | Glove `hand_skeleton`, subject profile, LeRobot release | **x, y, z, w** |

- **Meaning**: converted to a matrix `R`, the quaternion is the standard **active, local → world** rotation (column vectors); the **columns** of `R` are the tracker's own three axes in the world.
- **Fixed mount** (profile v5, `column: tracker_world = joint_world @ tracker_in_joint`):
  ```text
  R_tracker_world = R_joint_world · R_tracker_in_joint
  p_tracker_world = p_joint_world + R_joint_world · a_tracker_in_joint      (a is metric and is not scaled again)
  ```
  `attachment_position` / `attachment_rotation` are `a` and `R_tracker_in_joint`.
- **Relative motion**: `ΔR = R_0ᵀ · R_t`.
- **Change of world basis** `Q`: position `Q · p`, rotation `Q · R`.
- **6D rotation** (LeRobot `rotation_6d`): the first two matrix columns, order `r00, r10, r20, r01, r11, r21`.

## 11.3 Roles, anchor joints and left/right

| Tracker role | SMPL-X anchor joint |
|---|---|
| `camera` | 15 head |
| `left_wrist` / `right_wrist` | 20 / 21 |
| `left_ankle` / `right_ankle` | 7 / 8 |

`left_*` is always the **subject's own** left, never the image left. Front views are not mirrored: facing the subject, their left appears on the right of the image. No code or viewer may swap roles to "fix" a display.

Watch the order of `tracker_roles`: the raw npz and LeRobot use `camera, left_ankle, left_wrist, right_ankle, right_wrist` (alphabetical), unified_30hz and the profile use `camera, left_wrist, right_wrist, left_ankle, right_ankle`. **Index by name, not by position.**

## 11.4 SMPL-X

- Model: neutral SMPL-X, `num_betas = 10`, `use_pca = False` (45-D axis-angle hands).
- Body axes with no root rotation: **+X = body left, +Y = up, +Z = body forward**.
- `global_orient` is the world-relative root rotation, `body_pose` the local axis-angle of 21 joints (rad), `left/right_hand_pose` the local axis-angle of the 15 MANO joints.
- **Translation in unified_30hz (and the mini release's `smplx_episode.npz`)**:
  ```text
  V_world = scale · SMPLX(betas, global_orient, body_pose, hands, transl = 0).vertices + transl
  ```
  Tracker positions share the world frame and units of `V_world`; trackers are not multiplied by `scale`.
- **World pelvis position** `u_t` (the translation field of the paper / full release): SMPL-X rotates the root about the pelvis joint, so the pelvis position does not depend on pose:
  ```text
  u_t = transl_t + scale · J_pelvis(betas)          (J_pelvis = pelvis joint at zero pose and zero translation)
  ```

## 11.5 Gloves and hands

- `glove_{left,right}_points` in unified_30hz are joint points in the **wrist-local frame**. MANUS uses its own wrist localisation `R_wristᵀ (p − p_wrist)`; this is a different convention from the tracker mounting extrinsics and must not be mixed with it.
- Wuji skeletons are already wrist-local; the root quaternion is always identity and carries no global hand orientation. The hand's world orientation comes entirely from the wrist tracker and the calibrated wrist transform.
- SMPL-X hands are MANO joint for joint (left 25–39, right 40–54, ordered index / middle / pinky / ring / thumb). **The left hand is a mirror of the right**, so any direction from a cross product (flexion axis, palm normal) flips sign between the hands and must be derived per side from the rest pose. With `fwd = wrist → MCP centre` and `lat = pinky → index`, the palm direction is `+cross(fwd, lat)` on the left and `−cross(fwd, lat)` on the right.

## 11.6 Camera

- The GoPro is rigidly coupled to the head tracker, so camera pose = head tracker pose · a fixed camera–tracker extrinsic. That extrinsic **has not been calibrated separately yet** (`not_available` in the release).
- Released video: the raw 1920×1080 with the top 110 px cropped, giving 1920×970; intrinsics `fx = 1012.0, fy = 1016.0, cx = 960.0, cy = 430.0` (pixels).

## 11.7 Z-up conversion for display

For Z-up with the floor at 0:

```text
[x_view, y_view, z_view] = [x, −z, y − ground_y_m]       rotation [[1,0,0],[0,0,−1],[0,1,0]], det = +1
```

**Swapping Y and Z without the minus sign is a reflection (det = −1) and mirrors the body**, which is exactly what early viewers got wrong.

## 11.8 Time

- Raw files: microseconds, host wall clock `t_host_us`.
- `timestamps` in `smplx_profile_driven.npz`: **relative to the first sample** (start at 0); `absolute = relative + tracker_t_host[0]`.
- unified_30hz: absolute host wall clock (s) on integer multiples of 1/30 s.
- LeRobot: episode-relative seconds equal to the mp4 pts; `metadata.json` has `start_timestamp_host_us` to go back to absolute time.

## 11.9 Past mistakes (do not repeat)

1. Recorder quaternions were once treated as right-acting matrices and relative pose written as `R_t · R_0ᵀ`, so "look left" read as negative. The root cause was that the source data is **wxyz**, not xyzw. After the 2026-09-02 fix the profile format became v5 (column convention); **v4 and older profiles are rejected and must be recalibrated**. The "right-acting `R_tracker = A_pre @ R_joint_world`" still described in the project README and `calibration_protocol.md` is the old v4 wording; this chapter is authoritative.
2. The pelvis yaw was once initialised from the head tracker's **absolute mounting orientation** without removing the head-strap mount rotation, so the initial heading was off by about 155° and the optimiser crossed the limbs to reach the endpoints. The initial yaw now comes from a horizontal Procrustes fit on labelled endpoints.
3. Viewers once reordered `(X, Y, Z)` as `(X, Z, Y)` (a reflection), mirroring the body.
