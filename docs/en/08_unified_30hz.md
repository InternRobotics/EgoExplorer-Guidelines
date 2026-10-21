# 8. Unified 30 Hz export

[Back to contents](../index.md#english) · [中文](../zh/08_unified_30hz.md)

## 8.1 Why a unified export

No two streams in a raw recording share a rate: trackers at 65 or 90 Hz, gloves at 120 or 220 Hz, video at 30 or 60 Hz. The projected SMPL-X uses the tracker's own timestamps, counted from 0, while every other stream uses the host's absolute time. Lining these arrays up by index gives wrong results without any error.

`export_unified_30hz.py` writes one npz per recording in which **element i of every per-frame array is the same instant**. The time grid sits on whole multiples of 1/30 second of absolute time, so the same instant has the same index in every recording.

## 8.2 Running it

```bash
python scripts/export_unified_30hz.py --projected-root $WORK/out/projected \
  --output-root $WORK/out/unified_30hz --rate 30 --skip-existing
```

- It reads `smplx_with_manus_hands.npz` when hand fusion exists and `smplx_profile_driven.npz` otherwise (neutral hands).
- `--session <recording>` exports only that recording; `--keep-raw-tactile` also saves the raw values of all 744 taxels (about 17 MB per hand per recording).
- Wuji touch needs the processing modules in `tactile/` (`wuji_correct.py`, `wuji_layout.py`) and the glove health table (`health/health_db.npz`). Set `EGOEXPLORER_TACTILE_DIR` to use another location. Without them only raw values are exported, and the report says so.

Recordings whose projection failed can still get a video and touch export, used by the review pages of chapter 9 to show the cause. **Write it to a separate folder**: this script deletes the export of any projected recording found in its output folder.

```bash
python scripts/export_unified_nobody.py --sync-root $WORK/input/ego_data_sync \
  --batch-root $WORK/out --output-root $WORK/out/unified_30hz_nobody --skip-existing
```

## 8.3 How each signal is resampled

| Signal | Method | Why |
|---|---|---|
| Rotation | Slerp | Linear interpolation of axis-angle or rotation matrices is wrong |
| Position | Linear interpolation | |
| Validity flags | AND over the window | |
| Touch | Maximum over the window; OR for contact | Sampling a 120 Hz contact signal at 30 Hz would miss most brief contacts |
| Video | Index of the nearest frame, plus its offset from the grid time | Video frames cannot be interpolated |

Each tracker is resampled on its own, from its own valid samples. Grid points outside a tracker's coverage get `tracker_valid=False` instead of a forced value (zero-length quaternions make Slerp fail). Every stream also has an `*_interpolated` flag, True when a grid point is more than one native sample period from the nearest real sample, which separates measurements from values bridged across a dropout.

Round-trip error (resample to 30 Hz, then back to the original timestamps) stays within 0.1° / 0.5 mm with every touch peak kept, far below the fitting error itself.

## 8.4 Contents of `unified_30hz/<recording>.npz`

**Time**

| Key | Shape | Meaning |
|---|---|---|
| `timestamps` | (T,) float64 | Host absolute time (s) |
| `rate_hz` | () | 30.0 |
| `upsample_ratio` | () | 30 ÷ native tracker rate, for example about 0.46 on Wuji. Above 1, part of the values are interpolated |

**Body and hands** (neutral SMPL-X, `num_betas=10`, `use_pca=False`)

| Key | Shape | Meaning |
|---|---|---|
| `betas`, `scale` | (10,), () | From the calibration |
| `global_orient`, `body_pose`, `transl` | (T,3), (T,63), (T,3) | Axis-angle (rad); for `transl` see 11.4 |
| `left_hand_pose`, `right_hand_pose` | (T,45) | Axis-angle of the 15 MANO joints |
| `hand_pose_source` | string | `glove_fused` measured by the glove; `none_neutral_zero` placeholder |
| `attachment_position`, `attachment_rotation` | (5,3), (5,3,3) | Mounting translation and rotation of the five trackers relative to their joints |
| `body_interpolated` | (T,) bool | |
| `has_body` | () bool | False for exports of failed projections |
| `ground_y_m` | () | Floor height (Y) in raw coordinates |
| `coordinate_frame` | string | `raw_capture_y_up` |

**Trackers**

| Key | Shape | Meaning |
|---|---|---|
| `tracker_roles` | (5,) | `camera, left_wrist, right_wrist, left_ankle, right_ankle`, a different order from the raw file |
| `tracker_position`, `tracker_rotation` | (T,5,3), (T,5,3,3) | Position (m); rotation matrices whose columns are the tracker's own axes in the world |
| `tracker_valid`, `tracker_interpolated`, `tracker_native_hz` | | |

**Gloves**

| Key | Shape | Meaning |
|---|---|---|
| `glove_layout` | string | `wuji21` for 21 nodes (the value for both MANUS and Wuji) |
| `glove_left_points`, `glove_right_points` | (T,21,3) | Joint points in the wrist's local frame |
| `glove_*_native_hz`, `glove_*_interpolated` | | |

**Touch (Wuji only)**, `{S}` is `L` or `R`:

| Key | Shape | Meaning |
|---|---|---|
| `tactile_{S}_L4_contact` | (T,17) | Contact (0/1) for 17 regions, OR over the window |
| `tactile_{S}_L2_score` | (T,17) | Score after baseline removal, divided by a robust standard deviation |
| `tactile_{S}_L1_residual` | (T,17) | Residual after removing a rolling 10th-percentile baseline |
| `tactile_{S}_coarse_contact`, `_coarse_score` | (T,6) | Thumb, index, middle, ring, little finger, palm |
| `tactile_{S}_coverage`, `_sigma` | | Share of usable taxels per region, noise scale |
| `tactile_{S}_L0_raw` | (T,744) | Only with `--keep-raw-tactile` or without a health table |
| `tactile_region_names`, `tactile_coarse_names`, `tactile_levels` | | Region names and level definitions |

The 17 regions: `thumb_distal, thumb_proximal, index_{distal,middle,proximal}, middle_{…}, ring_{…}, pinky_{…}, palm_radial, palm_central, palm_ulnar`.

The touch levels and their thresholds are defined at the top of `wuji_correct.py`: L0 raw values; L1 rolling baseline removed; L2 divided by a robust standard deviation; L3 gain correction (ablation only, not exported); L4 contact from the regional 80th percentile with hysteresis. On ageing gloves, what recovers the data is looking at change instead of absolute values (L1, L2); gain correction helps little. Glove health is estimated per recording, never averaged per day, because many lines come and go.

**Video**

| Key | Shape | Meaning |
|---|---|---|
| `video_frame_index` | (T,) int32 | Index of the nearest frame |
| `video_time_offset_s` | (T,) | Actual offset of that frame from the grid time; measured maximum about 22 ms |
| `video_native_hz` | () | |

The `.json` of the same name is a summary: frame count, native rates, `body_interpolated_frac`, `tracker_valid_frac`, `hand_pose_source`, `video_max_offset_ms` and so on.

## 8.5 Things to know

- `timestamps` in `smplx_profile_driven.npz` start at 0; add `tracker_t_host[0]` to get absolute time. The unified export has already done this.
- Touch sides are taken from the device name, `glove_0` as L and `glove_1` as R, not from `slot` in a corrected copy. Every Wuji rig so far wears `glove_0` on the left, and every glove side correction so far was on MANUS data without touch, so nothing is affected. If a Wuji recording ever has its sides swapped, this needs to read `slot`.
