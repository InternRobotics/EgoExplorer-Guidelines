# EgoExplorer Guidelines

**中文** | [English](#english)

EgoExplorer 采集与重建全流程指南：从硬件、穿戴、主机设置、采集、标定，到原始数据 → SMPL-X → 30 Hz 统一导出 → 三联校验 → LeRobot v3.0。内容依据生产批次实际使用的代码和配置整理，整条处理管线在一台机器上就能跑完；所有参数都是生产批次（v3）真正在用的取值。

> 网页版：https://internrobotics.github.io/EgoExplorer-Guidelines/ 。采购清单和 3D 打印件待补充。

## 目录

| # | 中文 | English |
|---|---|---|
| 0 | [总览与数据流](docs/zh/00_overview.md) | [Overview and data flow](docs/en/00_overview.md) |
| 1 | [硬件组成与穿戴](docs/zh/01_hardware_and_wearing.md) | [Hardware and wearing](docs/en/01_hardware_and_wearing.md) |
| 2 | [采集主机设置](docs/zh/02_host_setup.md) | [Capture host setup](docs/en/02_host_setup.md) |
| 3 | [采集脚本](docs/zh/03_capture_scripts.md) | [Capture scripts](docs/en/03_capture_scripts.md) |
| 4 | [如何采集数据](docs/zh/04_data_collection.md) | [Collecting data](docs/en/04_data_collection.md) |
| 5 | [标定与标定数据采集](docs/zh/05_calibration.md) | [Calibration and calibration recordings](docs/en/05_calibration.md) |
| 6 | [原始数据格式](docs/zh/06_raw_data_format.md) | [Raw data format](docs/en/06_raw_data_format.md) |
| 7 | [原始数据 → SMPL-X（含手部融合、重训练）](docs/zh/07_smplx_reconstruction.md) | [Raw → SMPL-X (hands, retraining)](docs/en/07_smplx_reconstruction.md) |
| 8 | [30 Hz 统一导出](docs/zh/08_unified_30hz.md) | [Unified 30 Hz export](docs/en/08_unified_30hz.md) |
| 9 | [质检与三联校验](docs/zh/09_review_and_qc.md) | [QC and three-pane review](docs/en/09_review_and_qc.md) |
| 10 | [转换为 LeRobot v3.0](docs/zh/10_lerobot_v3.md) | [Converting to LeRobot v3.0](docs/en/10_lerobot_v3.md) |
| 11 | [坐标系约定](docs/zh/11_coordinate_systems.md) | [Coordinate conventions](docs/en/11_coordinate_systems.md) |
| 12 | [运行环境与常见坑](docs/zh/12_environment_and_pitfalls.md) | [Environment and pitfalls](docs/en/12_environment_and_pitfalls.md) |

## 一页速查

```text
穿戴 5×VIVE Ultimate Tracker + 手套(MANUS / Wuji) + GoPro HERO13(头部，与头部 tracker 刚性相连)
  └─ 每次穿戴：采一段 01–17 标定动作 → calibrate-subject → subject_profile.json
采集  *.tracker.npz / *.npz(手套) / *.video.npz / *.mp4 / *.session.json
重建  检查 + 左右修正 → 投影 → 手部融合 → 30 Hz 导出 → 三联校验页 → gallery（一台机器）
输出  unified_30hz/<session>.npz（所有流在同一条 30 Hz 绝对时间栅格上）
发布  unified_30hz + mp4 + 标注 → LeRobot v3.0（30 Hz，xyzw 四元数，+Y 向上）
```

---

<a id="english"></a>

# EgoExplorer Guidelines (English)

An end-to-end guide to the EgoExplorer capture and reconstruction workflow: hardware, wearing, host setup, capture, calibration, then raw data → SMPL-X → unified 30 Hz export → three-pane review → LeRobot v3.0. Everything here is taken from the code and configuration of our production (v3) batch, and the whole processing pipeline runs on one machine.

> Website: https://internrobotics.github.io/EgoExplorer-Guidelines/ . The bill of materials and 3D-printed parts are still to be added.

See the table of contents above; the English chapters live under [`docs/en/`](docs/en/).

## One-page cheat sheet

```text
Wear     5× VIVE Ultimate Tracker + gloves (MANUS / Wuji) + GoPro HERO13 (on the head, rigidly coupled to the head tracker)
  └─ per donning: record the 01–17 calibration protocol → calibrate-subject → subject_profile.json
Capture  *.tracker.npz / *.npz (gloves) / *.video.npz / *.mp4 / *.session.json
Rebuild  checks + L/R fixes → projection → hand fusion → 30 Hz export → review pages → gallery (one machine)
Output   unified_30hz/<session>.npz (every stream on one absolute 30 Hz grid)
Release  unified_30hz + mp4 + annotations → LeRobot v3.0 (30 Hz, xyzw quaternions, +Y up)
```
