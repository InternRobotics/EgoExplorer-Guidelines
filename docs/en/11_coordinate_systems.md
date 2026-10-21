# 11. Coordinate conventions

[Back to contents](../index.md#english) · [中文](../zh/11_coordinate_systems.md)

These are the project's coordinate conventions; the code is in `src/five_tracker_smplx/coordinates.py`. Read the whole chapter before reading or writing data; 11.9 lists the most common mistakes.

## 11.1 World frame

| Item | Convention |
|---|---|
| Source | The VIVE tracking system's world frame, stored as recorded with no translation, rotation or axis change. Labelled `raw_capture_y_up` in `unified_30hz` and `sdk-world-as-recorded` in LeRobot |
| Handedness and units | Right-handed, metres |
| Vertical | **+Y up**; gravity is `(0, −1, 0)` |
| Origin | The reference point set in the VIVE play-area map, neither on the floor nor on the wearer |
| Floor | At height `ground_y_m` (a Y value in raw coordinates), estimated by default from a low percentile of the ankle trackers |
| Horizontal heading | During calibration, floor tape places the opening stance so that **+X is the wearer's left and +Z the wearer's front**. Task recordings can start facing any direction; the body's left and front then come from the solved root rotation, and the world X and Z axes must not be read as the body's current left and front |

## 11.2 Tracker poses

**Position**: the tracker origin in the world frame, in metres.

**Quaternion order**: always **w, x, y, z**.

**Meaning of the rotation**: converted to a matrix `R`, the quaternion rotates from the tracker's local frame to the world frame (acting on column vectors); the **columns** of `R` are the tracker's own three axes in the world.

**Mounting** (calibration file format v5, `column: tracker_world = joint_world @ tracker_in_joint`):

```text
R_tracker_world = R_joint_world · R_tracker_in_joint
p_tracker_world = p_joint_world + R_joint_world · a_tracker_in_joint      (a is in metres and not scaled again)
```

`attachment_position` and `attachment_rotation` in the exports are `a` and `R_tracker_in_joint`.

Other relations:

- relative motion: `ΔR = R_0ᵀ · R_t`;
- change of world basis `Q`: positions become `Q · p`, rotations `Q · R`;
- 6D rotation (`rotation_6d` in LeRobot): the first two matrix columns, in the order `r00, r10, r20, r01, r11, r21`.

## 11.3 Roles, joints and sides

| Tracker role | SMPL-X joint |
|---|---|
| `camera` | 15, head |
| `left_wrist`, `right_wrist` | 20, 21 |
| `left_ankle`, `right_ankle` | 7, 8 |

`left_*` is always the wearer's own left. Front views are not mirrored: facing the wearer, their left appears on the right of the image. No code or visualisation may swap roles according to the image.

!!! warning "Role order differs between files"
    Raw npz files and LeRobot list roles alphabetically, `camera, left_ankle, left_wrist, right_ankle, right_wrist`; `unified_30hz` and the calibration file use `camera, left_wrist, right_wrist, left_ankle, right_ankle`. **Look roles up by name, never by position.**

## 11.4 SMPL-X

- Model: neutral SMPL-X, `num_betas = 10`, `use_pca = False` (45-value axis-angle hands).
- With no root rotation the body axes are **+X towards the body's left, +Y up, +Z towards the body's front**.
- `global_orient` is the root's rotation relative to the world, `body_pose` the local axis-angle of 21 joints (rad), and `left_hand_pose`, `right_hand_pose` the local axis-angle of the 15 MANO joints.

**Translation in `unified_30hz` and `smplx_episode.npz`**:

```text
V_world = scale · SMPLX(betas, global_orient, body_pose, hands, transl = 0).vertices + transl
```

Tracker positions and `V_world` share one world frame and one unit; trackers are not multiplied by `scale`.

**Pelvis position in world coordinates** `u_t` (the translation field of the paper and the full release). SMPL-X rotates the root about the pelvis joint, so the pelvis position does not depend on pose:

```text
u_t = transl_t + scale · J_pelvis(betas)          (J_pelvis: pelvis joint position at zero pose and zero translation)
```

## 11.5 Gloves and hands

- `glove_left_points` and `glove_right_points` in `unified_30hz` are joint points in the **wrist's local frame**. MANUS uses its own wrist-local formula `R_wristᵀ (p − p_wrist)`; this is a different convention from tracker mounting, and the two must not be mixed.
- Wuji skeletons are already wrist-local, with an identity root quaternion and no overall hand orientation. The hand's orientation in the world comes entirely from the wrist tracker and the calibrated wrist mounting.
- SMPL-X hands are MANO, joint for joint: joints 25–39 on the left and 40–54 on the right, in the order index, middle, little, ring, thumb.
- **The left hand mirrors the right.** Any direction obtained from a cross product (bending axis, palm normal) has opposite signs on the two hands and must be derived separately from the rest pose. With `fwd` from the wrist to the centre of the knuckles and `lat` from the little finger to the index finger, the palm faces `+cross(fwd, lat)` on the left hand and `−cross(fwd, lat)` on the right.

## 11.6 Camera

- The GoPro is rigidly coupled to the head tracker, so the camera pose is the head tracker pose times a fixed camera-to-tracker transform. That transform **has not been calibrated separately yet** and is marked `not_available` in the released data.
- Released video is the original 1920×1080 with the top 110 pixels cropped, giving 1920×970, with intrinsics `fx = 1012.0, fy = 1016.0, cx = 960.0, cy = 430.0` (pixels).

## 11.7 Z-up for display

For visualisation with Z up and the floor at 0:

```text
[x', y', z'] = [x, −z, y − ground_y_m]       rotation matrix [[1,0,0],[0,0,−1],[0,1,0]], determinant +1
```

Swapping Y and Z without the minus sign is a mirror (determinant −1) and flips the body left to right.

## 11.8 Time

| Data | Time convention |
|---|---|
| Raw files | Microseconds, host system clock `t_host_us` |
| `timestamps` in `smplx_profile_driven.npz` | Counted from the first frame (starting at 0); absolute time = relative time + `tracker_t_host[0]` |
| `unified_30hz` | Host absolute time (s), on whole multiples of 1/30 s |
| LeRobot | Seconds within the episode, equal to the mp4 display time; `start_timestamp_host_us` in `metadata.json` converts back to absolute time |

## 11.9 Common mistakes

1. **Reading quaternions in the wrong order.** With the order wrong, relative rotations come out reversed, for example "look left" becomes a turn to the right. Read them as w, x, y, z and compute relative motion as `ΔR = R_0ᵀ · R_t`.
2. **Using the head tracker's absolute orientation as the body's heading.** The head tracker's orientation includes the head strap's mounting rotation; using it directly leaves the initial heading far off, and the optimiser crosses the limbs to reach the trackers. Remove the mounting rotation first, or estimate the heading from the relative positions of the five trackers.
3. **Only swapping Y and Z for display.** That is a mirror and flips the body left to right; use the rotation in 11.7.
