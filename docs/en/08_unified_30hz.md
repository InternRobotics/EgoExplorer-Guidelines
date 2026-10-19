# 8. Unified 30 Hz export

[Back to contents](../../README.md#english) · [中文](../zh/08_unified_30hz.md)

## 8.1 Why a unified export

No two streams in a raw recording share a rate: trackers at 15 or 65 Hz, gloves at 22/120/220 Hz, video at 30–60 Hz. The SMPL-X result is written on the tracker's own timestamps, **relative to its first sample**, while every other stream is **absolute host wall clock**. Zipping these arrays by index is silently wrong.

`export_unified_30hz.py` writes one npz per session in which **element i of every per-frame array is the same instant**. The grid is anchored to integer multiples of 1/30 s of absolute time, so the same instant has the same index in any session.

## 8.2 Running it

```bash
bash scripts/run_batch_v3.sh export
# equivalent to:
$PYCPU scripts/export_unified_30hz.py --projected-root $BATCH_ROOT/projected \
  --output-root $BATCH_ROOT/unified_30hz --rate 30 --skip-existing
```

- Optional `--session <sid>` (repeatable) and `--keep-raw-tactile` (also store the window maximum of the 744 raw taxels, about 17 MB per hand per session).
- Reads the fused `smplx_with_manus_hands.npz` when present, otherwise `smplx_profile_driven.npz` (hands as a neutral zero-pose placeholder).
- Sessions whose projection failed are exported with `export_unified_nobody.py` (video and touch, no body), with the failure reason stored so the review page can show it.
- Wuji touch needs `~/workspace/tactile_work/` (`wuji_correct.py`, `wuji_layout.py`, `health/health_db.npz`). Without it only L0 raw values are exported and the report says "L0 only".

## 8.3 Resampling rules

| Signal | Method | Why |
|---|---|---|
| Rotations | Slerp | Linear interpolation of axis-angle or matrices is wrong |
| Positions | Linear | |
| Validity masks | Logical AND over the window | |
| Touch | **Window maximum / logical OR for contact** | Point-sampling a 120 Hz contact signal at 30 Hz drops most brief touches |
| Video | Nearest frame index, plus its offset from the grid instant | Frames cannot be interpolated |

Each tracker is resampled **independently** from its own valid samples; grid points outside a tracker's coverage get `tracker_valid=False` instead of a forced interpolation (zero-norm quaternions crash Slerp). Every stream also gets an `*_interpolated` mask, True where a grid point is more than one native period from the nearest real sample, so a measurement can be told apart from a bridge across a dropout.

Round-trip check (30 Hz and back to native timestamps): MANUS 0.069° / 0.50 mm, Wuji 0.016° / 0.11 mm, 100% of touch peaks kept, all far below the fitting residual itself.

## 8.4 Output keys (`unified_30hz/<session>.npz`)

**Timeline**

| Key | Shape | Meaning |
|---|---|---|
| `timestamps` | (T,) float64 | Absolute host wall clock (s) |
| `rate_hz` | () | 30.0 |
| `upsample_ratio` | () | 30 ÷ native tracker rate. MANUS ≈ 1.98 (half of the samples are interpolated), Wuji ≈ 0.46 (downsampled) |

**Body + hands SMPL-X** (neutral SMPL-X, `num_betas=10`, `use_pca=False`)

| Key | Shape | Meaning |
|---|---|---|
| `betas` / `scale` | (10,) / () | From the subject profile |
| `global_orient` / `body_pose` / `transl` | (T,3) / (T,63) / (T,3) | Axis-angle (rad); for `transl` see 11.4 |
| `left_hand_pose` / `right_hand_pose` | (T,45) | MANO 15-joint axis-angle |
| `hand_pose_source` | string | `glove_fused` (measured) or `none_neutral_zero` (placeholder, not a measurement) |
| `attachment_position` / `attachment_rotation` | (5,3) / (5,3,3) | Mounting translation / rotation of the five trackers in their joint frames |
| `body_interpolated` | (T,) bool | |
| `has_body` | () bool | False for exports of failed projections |
| `ground_y_m` | () | Floor height (Y) in raw coordinates |
| `coordinate_frame` | string | `raw_capture_y_up` |

**Trackers**

| Key | Shape | Meaning |
|---|---|---|
| `tracker_roles` | (5,) | `camera, left_wrist, right_wrist, left_ankle, right_ankle` (note: a different order from the raw npz) |
| `tracker_position` / `tracker_rotation` | (T,5,3) / (T,5,3,3) | Metres / rotation matrices (columns = the tracker's own axes in the world) |
| `tracker_valid` / `tracker_interpolated` / `tracker_native_hz` | | |

**Gloves**

| Key | Shape | Meaning |
|---|---|---|
| `glove_layout` | string | `manus25` or `wuji21` |
| `glove_left_points` / `glove_right_points` | (T,N,3) | Joint points in the **wrist-local frame**, N = 25 or 21 |
| `glove_*_native_hz` / `glove_*_interpolated` | | |

**Touch (Wuji only)**, `{S}` ∈ `L`, `R`:

| Key | Shape | Meaning |
|---|---|---|
| `tactile_{S}_L4_contact` | (T,17) | Binary contact per region (logical OR over the window) |
| `tactile_{S}_L2_score` | (T,17) | Robust z-score (divided by MAD) |
| `tactile_{S}_L1_residual` | (T,17) | Residual after removing a rolling p10 baseline |
| `tactile_{S}_coarse_contact` / `_coarse_score` | (T,6) | Thumb / index / middle / ring / little / palm |
| `tactile_{S}_coverage` / `_sigma` | | Usable-taxel fraction per region / noise scale |
| `tactile_{S}_L0_raw` | (T,744) | Only with `--keep-raw-tactile` or without a health table |
| `tactile_region_names` / `tactile_coarse_names` / `tactile_levels` | | Names and level definitions |

The 17 regions: `thumb_distal, thumb_proximal, index_{distal,middle,proximal}, middle_{…}, ring_{…}, pinky_{…}, palm_radial, palm_central, palm_ulnar`.

Touch levels (defined in `wuji_correct.py`, thresholds at the top of the file): L0 raw → L1 rolling-p10 baseline removed → L2 divided by a robust MAD to give a z-score → L3 gain correction (ablation only, not released) → L4 region 80th percentile + a hysteretic dual criterion for contact. **What rescues data from a degrading glove is using change instead of absolute value (L1/L2), not gain correction.** Health is estimated per session, never averaged per day (many lines are intermittent). Older exports carried per-finger means named `tactile_L0…L4`; those are not the same thing as the L0–L4 levels here.

**Video**

| Key | Shape | Meaning |
|---|---|---|
| `video_frame_index` | (T,) int32 | Nearest video frame |
| `video_time_offset_s` | (T,) | Actual offset of that frame from the grid instant (measured max about 22 ms) |
| `video_native_hz` | () | |

The `<session>.json` next to it is a summary: frame count, native rates, `body_interpolated_frac`, `tracker_valid_frac`, `hand_pose_source`, `video_max_offset_ms`, etc.

## 8.5 Things to watch

- Never use the `timestamps` of `smplx_profile_driven.npz` as absolute time: they start at 0, and `body_abs = body_rel + tracker_t_host[0]`. The unified export has already converted them.
- Touch sides are taken from the device name (`glove_0 → L`, `glove_1 → R`), not from the `slot` in a corrected copy. Every Wuji rig so far wears glove_0 on the left, and every glove-side correction so far was on MANUS data without touch, so nothing is affected today; if a Wuji slot is ever swapped, this needs to read `slot`.
