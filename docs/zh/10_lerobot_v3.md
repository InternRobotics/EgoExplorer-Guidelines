# 10. 转换为 LeRobot v3.0

[返回目录](../index.md) · [English](../en/10_lerobot_v3.md)

发布数据采用 [LeRobotDataset v3.0](https://github.com/huggingface/lerobot) 格式。转换脚本是工具包里的 `scripts/build_lerobot_dataset.py`，输入就是前面各章的产出：原始录制和 `unified_30hz/`。

![步骤标注与第一视角时间轴对齐](../assets/img/text_annotations.jpg)

*每条录制带有步骤级标注：步骤编号、描述、起止时间、技能、物体角色和结果。*

## 10.1 目录结构

```text
<dataset>/
├── meta/
│   ├── info.json                              fps=30、各字段的类型与形状、路径模板、坐标系说明
│   ├── stats.json                             各字段的统计量
│   ├── tasks.parquet                          task_index 与任务描述的对应
│   └── episodes/chunk-000/file-000.parquet    每个 episode 的行范围、视频偏移
├── data/chunk-000/file-NNN.parquet            逐帧数据
├── videos/observation.images.camera_0/chunk-000/file-NNN.mp4
└── episode_meta/episode_XXXXXX/               每个 episode 的附加信息（XXXXXX = episode_index + 1）
    ├── metadata.json                          录制名、任务、场景、起止时间、设备清单、裁剪方式、处理说明
    ├── calibration/calibration.json           相机内参、tracker 安装参数、坐标系、触觉标定、时钟信息
    ├── annotations/instruction.json           任务指令
    ├── annotations/segments.parquet           步骤标注
    ├── quality/quality.json                   拟合残差、tracker 插值统计
    └── smplx/smplx_episode.npz                该 episode 的 SMPL-X（约定与 unified_30hz 相同）
```

## 10.2 逐帧字段

| 字段 | 类型与形状 | 说明 |
|---|---|---|
| `timestamp` `frame_index` `episode_index` `index` `task_index` | 标量 | `timestamp` 是 episode 内的秒数，与 mp4 的显示时间一致 |
| `tracker.<role>.position` | float32 (3) | 米；`<role>` 为 camera、left_wrist、right_wrist、left_ankle、right_ankle |
| `tracker.<role>.orientation` | float32 (4) | 四元数，顺序为 **x, y, z, w**（原始文件是 wxyz） |
| `tracker.<role>.rotation_6d` | float32 (6) | 旋转矩阵的前两列，顺序为 `r00, r10, r20, r01, r11, r21` |
| `gravity.direction` | float32 (3) | `(0, -1, 0)` |
| `observation.hand.{left,right}.joint_positions` | float32 (21, 3) | 手套关节点（21 节点） |
| `observation.hand.{side}.tactile_17zone_contact` | float32 (17) | 各区域接触（0 或 1） |
| `observation.hand.{side}.tactile_17zone_score` | float32 (17) | 去基线后的分数 |
| `observation.hand.{side}.tactile_17zone_residual` | float32 (17) | 基线残差 |
| `observation.hand.{side}.tactile_17zone_coarse_contact`、`_coarse_score` | float32 (6) | 五指加手掌 |
| `observation.images.camera_0` | 视频 (970, 1920, 3) | 裁掉顶部 110 像素，严格 30 fps |

没有触觉的手（例如 MANUS 录制），触觉字段为 NaN。

## 10.3 运行转换

**1. 准备 episode 清单**：CSV 文件，一行一个 episode，行的顺序就是 episode 的顺序。

```csv
session,task_zh,task_en,environment_type,trim_head_s,trim_tail_s
session_20260901_154755_物料称量,物料称量,Weighing,lab,0,5
session_20260916_112944_萃取,萃取,Extraction,lab,0,0
```

`environment_type` 填 `lab` 或 `home`；`trim_head_s` 和 `trim_tail_s` 可选，用于去掉开头和结尾。

**2. 运行**

```bash
python scripts/build_lerobot_dataset.py \
  --episodes episodes.csv \
  --raw-root $WORK/input/ego_data_sync \
  --unified-root $WORK/out/unified_30hz \
  --projected-root $WORK/out/projected \
  --output $WORK/out/lerobot
```

| 参数 | 说明 |
|---|---|
| `--projected-root` | 可选，把拟合残差写进 `quality.json` |
| `--captions-root` | 可选，存放 `<录制名>.en.jsonl` 步骤标注的目录 |
| `--crop-top` | 视频顶部裁掉的像素，默认 110（去掉 GoPro 屏幕信息） |
| `--fx` `--fy` `--cx` `--cy` | 写进 `calibration.json` 的相机内参，默认 1012 / 1016 / 960 / 430 |
| `--license` `--privacy-status` | 写进元数据的许可和隐私说明 |
| `--validate-only` | 只检查已有的数据集，不重新转换 |

视频编码需要系统里有 `ffmpeg`。已经编码过的视频会跳过，中断后重跑即可接着做。

转换结束时会自动检查：每个 episode 的 parquet 行数等于视频帧数，`timestamp` 落在 1/30 秒的栅格上（误差小于 1e-4 秒）。在一条 192 秒的 Wuji 录制上实测：Mac CPU 约 1 分钟，得到 5,620 帧，视频帧数与行数一致，栅格误差 7×10⁻⁶ 秒；用 `lerobot` 0.4.4 的 `LeRobotDataset` 以默认容差读取首、中、末三帧（含视频解码）均正常。

## 10.4 转换器做了什么

1. **重新编码视频。** 原始 mp4 是可变帧率，`timestamp` 和显示时间最多差约 25 ms，会让 LeRobot 默认 `tolerance_s = 1e-4` 的视频读取报错。因此用 ffmpeg 裁掉顶部 110 像素并转成严格 30 fps，之后第 k 帧的显示时间正好是 k/30 秒。
2. **以视频帧为时间轴。** tracker 从原始约 65 Hz 的数据插值到每个视频帧的时刻：位置线性插值，旋转用最短路径 Slerp。原始数据中断超过 0.25 秒，或者相邻两个样本位置跳变超过 0.10 m、旋转跳变超过 60°（tracker 重新定位或坏帧）时，不在这段之间插值，而是取较近一端的样本。每个 tracker 的插值统计写进 `quality.json`。
3. **四元数转为 xyzw**，同时写出 `rotation_6d`。
4. **手部与触觉**取自 `unified_30hz`。导出里缺失的手，用同一套触觉处理从原始手套文件重新计算，来源记在 `metadata.json` 和 `info.json` 里。某个区域没有可用触点时，分数和残差为 NaN，接触恒为 0，各区域覆盖率记在 `coverage_17`。
5. **SMPL-X** 每个 episode 存一份 `smplx_episode.npz`，键与 `unified_30hz` 相同；体型、尺度和安装参数写在 `calibration.json` 里，不在每一行重复。
6. **写入元数据**：`info.json`、`stats.json`、`tasks.parquet` 和 `episodes/…parquet`。

发布前，建议再用 `LeRobotDataset` 以默认容差完整读一遍每个 episode。

## 10.5 读取示例

```python
from lerobot.datasets.lerobot_dataset import LeRobotDataset

ds = LeRobotDataset("InternRobotics/EgoExplorer", root="EgoExplorer/dataset/lerobot")
print(ds.meta.info["fps"], ds.num_episodes, ds.num_frames)
frame = ds[0]
frame["observation.images.camera_0"]          # (3, H, W)
frame["tracker.camera.orientation"]           # x, y, z, w
```

SMPL-X 前向计算（与总览页、校验页的约定相同）：

```python
import numpy as np, smplx, torch

z = np.load("episode_meta/episode_000001/smplx/smplx_episode.npz")
model = smplx.create("models", model_type="smplx", gender="neutral", use_pca=False, num_betas=10)
T = len(z["body_pose"]); zero = torch.zeros(T, 3)
out = model(betas=torch.tensor(z["betas"])[None].expand(T, -1),
            global_orient=torch.tensor(z["global_orient"]), body_pose=torch.tensor(z["body_pose"]),
            left_hand_pose=torch.tensor(z["left_hand_pose"]), right_hand_pose=torch.tensor(z["right_hand_pose"]),
            transl=zero, jaw_pose=zero, leye_pose=zero, reye_pose=zero, expression=torch.zeros(T, 10))
vertices = float(z["scale"]) * out.vertices.numpy() + z["transl"][:, None]   # 与 tracker 同一世界坐标系，+Y 向上
```

## 10.6 正式发布前需要确定的事

- **平移字段的含义。** `smplx_episode.npz` 沿用 `unified_30hz` 的约定（顶点 = scale × SMPL-X(transl=0) + transl）；论文和 HF 数据卡描述的完整版里，平移字段存的是世界坐标下的骨盆位置（换算见第 11.4 节）。正式发布前要选定一种，并写进 `info.json`。
- **视频字段名。** 当前为 `observation.images.camera_0`，论文附录写的是 `observation.images.ego`。
