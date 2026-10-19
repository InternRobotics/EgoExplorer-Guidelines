# 10. 转换为 LeRobot v3.0

[返回目录](../../README.md) · [English](../en/10_lerobot_v3.md)

发布数据遵循 [LeRobotDataset v3.0](https://github.com/huggingface/lerobot)。HF 上的 mini release（`InternRobotics/EgoExplorer`，`dataset/lerobot/`，18 个 episode、130,590 帧、10 类任务、全部为 Wuji）就是按本章流程从 `unified_30hz` + 原始录制转出来的。转换脚本目前在 H 集群 `/home/fangkaipeng/workspace/lerobot/`（`build_dataset_new.py`，`converter_version 2.0.0`），整理后并入工具包。

## 10.1 目录结构

```text
<dataset>/
├── meta/
│   ├── info.json                         # codebase_version=v3.0, fps=30, features, 路径模板, 坐标系说明
│   ├── stats.json                        # 逐特征统计
│   ├── tasks.parquet                     # task_index → 任务描述
│   └── episodes/chunk-000/file-000.parquet   # 每个 episode 的行范围、视频偏移、session / split / profile ID
├── data/chunk-000/file-NNN.parquet       # 逐帧数值 + 文本标注
├── videos/observation.images.camera_0/chunk-000/file-NNN.mp4
└── episode_meta/episode_XXXXXX/          # 扩展：每个 episode 的附加信息（XXXXXX = episode_index + 1）
    ├── metadata.json                     # session、任务、场景、时间范围、传感器清单、裁剪、处理说明
    ├── calibration/calibration.json      # 相机内外参、tracker 安装变换、坐标系、触觉标定、同步偏移
    ├── annotations/{instruction.json, segments.parquet}   # 任务指令、步骤标注
    ├── quality/quality.json              # 残差、tracker 插值统计、标志位
    └── smplx/smplx_episode.npz           # 该 episode 的 SMPL-X（与 unified_30hz 同约定）
```

## 10.2 逐帧特征（mini release 实际 schema）

| 特征 | dtype / shape | 来源与说明 |
|---|---|---|
| `timestamp`, `frame_index`, `episode_index`, `index`, `task_index` | 标量 | `timestamp` 为 episode 内相对秒数，**等于 mp4 的 pts** |
| `tracker.<role>.position` | float32 (3) | 米；`<role>` ∈ camera / left_wrist / right_wrist / left_ankle / right_ankle |
| `tracker.<role>.orientation` | float32 (4) | 四元数 **x, y, z, w**（注意与原始 npz 的 wxyz 不同） |
| `tracker.<role>.rotation_6d` | float32 (6) | 旋转矩阵前两列，顺序 `r00, r10, r20, r01, r11, r21`，方向 tracker 局部 → 世界 |
| `gravity.direction` | float32 (3) | `(0, -1, 0)` |
| `observation.hand.{left,right}.joint_positions` | float32 (21, 3) | 手套关节点（21 点 Wuji 拓扑），来自 unified 栅格 |
| `observation.hand.{side}.tactile_17zone_contact` | float32 (17) | L4 接触 0/1 |
| `observation.hand.{side}.tactile_17zone_score` | float32 (17) | L2 稳健 z 分数 |
| `observation.hand.{side}.tactile_17zone_residual` | float32 (17) | L1 基线残差 |
| `observation.hand.{side}.tactile_17zone_coarse_contact` / `_coarse_score` | float32 (6) | 粗分区 |
| `observation.images.camera_0` | video (970, 1920, 3) | 顶部裁掉 110 px，严格 30 fps CFR |

论文 / HF 数据卡描述的完整版本还会加入：SMPL-X 身体（世界骨盆位置 + 根朝向 + 21 个关节轴角）、双手 2×45、步骤标注 `annotation.*`、逐流 validity 掩码与溯源字段。

## 10.3 转换步骤

1. **选 episode。** 候选 = 质量表里 status=ok ∩ 有步骤标注 ∩ 有 SMPL-X ∩（要发布触觉时）双手触觉齐全。按残差挑：例如 p50 ≈ 3–4 mm、p95 < 8 mm、平均旋转误差 < 7°。可以掐头去尾（`trim_head_s` / `trim_tail_s` 记进 metadata）。
2. **视频重编码为严格 30 fps CFR。** 原始 mp4 是 VFR，`timestamp` 与 pts 最多差约 25 ms，会触发 LeRobot 默认的 `tolerance_s = 1e-4` 读取报错。用 ffmpeg 做 `crop=1920:970:0:110, fps=30` 并加 `-fps_mode cfr`，之后 `timestamp = k/30` 与 pts 逐一对应（校验 `max|pts − timestamp| ≤ 1.5e-5 s`）。
3. **以视频帧时间轴为 episode 时间轴。** tracker 从原始 `*.tracker.npz`（约 65 Hz）在有效样本上插值到每个视频帧时刻：位置线性，姿态最短路径 Slerp。**保护规则**：原生丢帧 > 0.25 s，或相邻样本位置跳变 > 0.10 m、姿态跳变 > 60°（tracker 重定位或坏帧）时，不跨这段插值，而是保持较近的端点样本。每个 role 的插值统计写进 `quality.json → tracker_interpolation`。
4. **四元数转成 xyzw**，同时写 `rotation_6d`。
5. **手部与触觉**从 `unified_30hz` 取（重采样到视频时间轴）。导出里缺失的手，用同一条 `wuji_correct` L0→L4 管线从原始手套 npz 重建，并在 `metadata.json` / `info.json → tactile_processing.per_episode_hand_source` 里记录来源。没有健康 taxel 的区，score/residual 记 `NaN`、contact 恒为 0，逐区覆盖率记 `coverage_17`。
6. **SMPL-X** 每个 episode 存一个 `episode_meta/*/smplx/smplx_episode.npz`（键与 unified_30hz 一致），体型、尺度、挂点放 `calibration.json`，不在每行重复。
7. **写 meta**：`info.json`（features、`coordinate_system`、`tracker_processing`、`tactile_processing`、`converter_version`）、`stats.json`、`tasks.parquet`、`episodes/…parquet`。
8. **校验**：
   - 结构与数值检查（mini release 是 782 项全过）；
   - 用 `LeRobotDataset` 在**默认容差**下端到端读取每个 episode 的视频和数据。

## 10.4 读取示例

```python
from lerobot.datasets.lerobot_dataset import LeRobotDataset

ds = LeRobotDataset("InternRobotics/EgoExplorer", root="EgoExplorer/dataset/lerobot")
print(ds.meta.info["fps"], ds.num_episodes, ds.num_frames)
frame = ds[0]
frame["observation.images.camera_0"]          # (3, H, W)
frame["tracker.camera.orientation"]           # xyzw
```

SMPL-X 前向（与 gallery / 审阅页同约定）：

```python
import numpy as np, smplx, torch
z = np.load("episode_meta/episode_000001/smplx/smplx_episode.npz")
m = smplx.create("models", model_type="smplx", gender="neutral", use_pca=False, num_betas=10)
T = len(z["body_pose"]); zero = torch.zeros(T, 3)
out = m(betas=torch.tensor(z["betas"])[None].expand(T, -1),
        global_orient=torch.tensor(z["global_orient"]), body_pose=torch.tensor(z["body_pose"]),
        left_hand_pose=torch.tensor(z["left_hand_pose"]), right_hand_pose=torch.tensor(z["right_hand_pose"]),
        transl=zero, jaw_pose=zero, leye_pose=zero, reye_pose=zero, expression=torch.zeros(T, 10))
world = float(z["scale"]) * out.vertices.numpy() + z["transl"][:, None]   # 与 tracker 同一世界系（+Y 向上）
```

## 10.5 需要统一的地方

- **平移字段的约定**：mini release 的 `smplx_episode.npz` 沿用 unified_30hz 的约定（`world = scale · SMPL-X(transl=0) + transl`）；论文和 HF 数据卡描述的完整版本里，平移字段存的是**世界骨盆位置**，不是 SMPL-X 库的加性 `transl`（换算见第 11.4 节）。完整版发布前要定下来用哪一种，并写进 `info.json`。
- **视频键名**：mini release 用 `observation.images.camera_0`，论文附录写的是 `observation.images.ego`。
- 相机内参（1920×970：fx 1012.0、fy 1016.0、cx 960.0、cy 430.0）目前只写在数据卡和论文里，`calibration.json` 里是 `not_available`，可以补上。
