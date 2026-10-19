# 10. Converting to LeRobot v3.0

[Back to contents](../index.md#english) · [中文](../zh/10_lerobot_v3.md)

Released data follows [LeRobotDataset v3.0](https://github.com/huggingface/lerobot). The mini release on HF (`InternRobotics/EgoExplorer`, `dataset/lerobot/`: 18 episodes, 130,590 frames, 10 task templates, all Wuji) was converted from `unified_30hz` + the raw recordings following this chapter. The converter currently lives in the data team's lerobot workspace on the H cluster (`build_dataset_new.py`, `converter_version 2.0.0`) and will be folded into the toolkit.

## 10.1 Layout

```text
<dataset>/
├── meta/
│   ├── info.json                         # codebase_version=v3.0, fps=30, features, path templates, coordinate system
│   ├── stats.json                        # per-feature statistics
│   ├── tasks.parquet                     # task_index → task description
│   └── episodes/chunk-000/file-000.parquet   # per-episode row ranges, video offsets, session / split / profile IDs
├── data/chunk-000/file-NNN.parquet       # frame-level numbers + text annotations
├── videos/observation.images.camera_0/chunk-000/file-NNN.mp4
└── episode_meta/episode_XXXXXX/          # extension: per-episode extras (XXXXXX = episode_index + 1)
    ├── metadata.json                     # session, task, scene, time range, sensor inventory, crop, processing notes
    ├── calibration/calibration.json      # camera intrinsics/extrinsics, tracker mounts, coordinate convention, touch calibration, sync offsets
    ├── annotations/{instruction.json, segments.parquet}   # task instruction, step annotations
    ├── quality/quality.json              # residuals, tracker interpolation statistics, flags
    └── smplx/smplx_episode.npz           # SMPL-X for the episode (same convention as unified_30hz)
```

## 10.2 Per-frame features (actual mini-release schema)

| Feature | dtype / shape | Source and notes |
|---|---|---|
| `timestamp`, `frame_index`, `episode_index`, `index`, `task_index` | scalars | `timestamp` is episode-relative seconds and **equals the mp4 pts** |
| `tracker.<role>.position` | float32 (3) | Metres; `<role>` ∈ camera / left_wrist / right_wrist / left_ankle / right_ankle |
| `tracker.<role>.orientation` | float32 (4) | Quaternion **x, y, z, w** (unlike the raw npz's wxyz) |
| `tracker.<role>.rotation_6d` | float32 (6) | First two matrix columns, order `r00, r10, r20, r01, r11, r21`, tracker-local → world |
| `gravity.direction` | float32 (3) | `(0, -1, 0)` |
| `observation.hand.{left,right}.joint_positions` | float32 (21, 3) | Glove joints (21-node Wuji topology) from the unified grid |
| `observation.hand.{side}.tactile_17zone_contact` | float32 (17) | L4 contact 0/1 |
| `observation.hand.{side}.tactile_17zone_score` | float32 (17) | L2 robust z-score |
| `observation.hand.{side}.tactile_17zone_residual` | float32 (17) | L1 baseline residual |
| `observation.hand.{side}.tactile_17zone_coarse_contact` / `_coarse_score` | float32 (6) | Coarse zones |
| `observation.images.camera_0` | video (970, 1920, 3) | Top 110 px cropped, strict 30 fps CFR |

The full version described in the paper and the HF card adds: SMPL-X body (world pelvis position + root orientation + 21 joint axis-angles), hands 2×45, step annotations `annotation.*`, and per-stream validity masks and provenance fields.

## 10.3 Conversion steps

1. **Select episodes.** Candidates = status ok in the quality table ∩ has step annotations ∩ has SMPL-X ∩ (if touch is released) touch on both hands. Rank by residual, e.g. p50 ≈ 3–4 mm, p95 < 8 mm, mean rotation error < 7°. Heads and tails may be trimmed (record `trim_head_s` / `trim_tail_s` in the metadata).
2. **Re-encode video to strict 30 fps CFR.** The raw mp4 is VFR; `timestamp` and pts differ by up to ~25 ms, which breaks LeRobot's default `tolerance_s = 1e-4` on read. Use ffmpeg with `crop=1920:970:0:110, fps=30` and `-fps_mode cfr`; afterwards `timestamp = k/30` matches the pts one to one (check `max|pts − timestamp| ≤ 1.5e-5 s`).
3. **Use the video frame timeline as the episode timeline.** Interpolate trackers from the raw `*.tracker.npz` (~65 Hz) onto each video frame time using valid samples only: linear for position, shortest-path Slerp for orientation. **Guards**: across a native dropout > 0.25 s, or a position step > 0.10 m / orientation step > 60° between neighbouring samples (relocalisation or a bad frame), do not interpolate through the discontinuity; hold the nearer native sample. Write per-role interpolation statistics to `quality.json → tracker_interpolation`.
4. **Convert quaternions to xyzw** and also write `rotation_6d`.
5. **Hands and touch** come from `unified_30hz` (resampled to the video timeline). A hand missing from the export is rebuilt from the raw glove npz with the same `wuji_correct` L0→L4 pipeline, and its source is recorded in `metadata.json` / `info.json → tactile_processing.per_episode_hand_source`. Regions without healthy taxels get `NaN` score/residual and constant-0 contact; per-region coverage goes into `coverage_17`.
6. **SMPL-X** is stored once per episode in `episode_meta/*/smplx/smplx_episode.npz` (same keys as unified_30hz); shape, scale and mounts go into `calibration.json` instead of repeating in every row.
7. **Write meta**: `info.json` (features, `coordinate_system`, `tracker_processing`, `tactile_processing`, `converter_version`), `stats.json`, `tasks.parquet`, `episodes/…parquet`.
8. **Validate**:
   - structural and numerical checks (782 checks, all passing, for the mini release);
   - an end-to-end read of every episode's video and data through `LeRobotDataset` at the **default tolerance**.

## 10.4 Reading

```python
from lerobot.datasets.lerobot_dataset import LeRobotDataset

ds = LeRobotDataset("InternRobotics/EgoExplorer", root="EgoExplorer/dataset/lerobot")
print(ds.meta.info["fps"], ds.num_episodes, ds.num_frames)
frame = ds[0]
frame["observation.images.camera_0"]          # (3, H, W)
frame["tracker.camera.orientation"]           # xyzw
```

SMPL-X forward pass (same convention as the gallery and the review pages):

```python
import numpy as np, smplx, torch
z = np.load("episode_meta/episode_000001/smplx/smplx_episode.npz")
m = smplx.create("models", model_type="smplx", gender="neutral", use_pca=False, num_betas=10)
T = len(z["body_pose"]); zero = torch.zeros(T, 3)
out = m(betas=torch.tensor(z["betas"])[None].expand(T, -1),
        global_orient=torch.tensor(z["global_orient"]), body_pose=torch.tensor(z["body_pose"]),
        left_hand_pose=torch.tensor(z["left_hand_pose"]), right_hand_pose=torch.tensor(z["right_hand_pose"]),
        transl=zero, jaw_pose=zero, leye_pose=zero, reye_pose=zero, expression=torch.zeros(T, 10))
world = float(z["scale"]) * out.vertices.numpy() + z["transl"][:, None]   # same world frame as the trackers (+Y up)
```

## 10.5 Open items to settle before the full release

- **Translation convention**: the mini release's `smplx_episode.npz` keeps the unified_30hz convention (`world = scale · SMPL-X(transl=0) + transl`); the full version described in the paper and HF card stores the **world pelvis position** in the translation field instead of the SMPL-X library's additive `transl` (conversion in 11.4). Pick one before the full release and state it in `info.json`.
- **Video key**: the mini release uses `observation.images.camera_0`; the paper appendix says `observation.images.ego`.
- The camera intrinsics (1920×970: fx 1012.0, fy 1016.0, cx 960.0, cy 430.0) are only in the data card and paper; `calibration.json` says `not_available` and could carry them.
