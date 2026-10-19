# EgoExplorer Guidelines

**中文** | [English](#english)

EgoExplorer 采集与重建全流程指南：从硬件、穿戴、主机设置、采集、标定，到原始数据 → SMPL-X → 30 Hz 统一导出 → 三联校验 → LeRobot v3.0。内容依据实际使用的代码和配置整理，参数都是生产批次（v3）真正在用的取值，整条处理管线在一台机器上就能跑完。

> 状态：草稿。采购清单和 3D 打印件待补充。代码：[InternRobotics/egoexplorer-toolkit](https://github.com/InternRobotics/egoexplorer-toolkit)。

| # | 中文 | English |
|---|---|---|
| 0 | [总览与数据流](zh/00_overview.md) | [Overview and data flow](en/00_overview.md) |
| 1 | [硬件组成与穿戴](zh/01_hardware_and_wearing.md) | [Hardware and wearing](en/01_hardware_and_wearing.md) |
| 2 | [采集主机设置](zh/02_host_setup.md) | [Capture host setup](en/02_host_setup.md) |
| 3 | [采集脚本](zh/03_capture_scripts.md) | [Capture scripts](en/03_capture_scripts.md) |
| 4 | [如何采集数据](zh/04_data_collection.md) | [Collecting data](en/04_data_collection.md) |
| 5 | [标定与标定数据采集](zh/05_calibration.md) | [Calibration and calibration recordings](en/05_calibration.md) |
| 6 | [原始数据格式](zh/06_raw_data_format.md) | [Raw data format](en/06_raw_data_format.md) |
| 7 | [原始数据 → SMPL-X（含手部融合、重训练）](zh/07_smplx_reconstruction.md) | [Raw → SMPL-X (hands, retraining)](en/07_smplx_reconstruction.md) |
| 8 | [30 Hz 统一导出](zh/08_unified_30hz.md) | [Unified 30 Hz export](en/08_unified_30hz.md) |
| 9 | [质检与三联校验](zh/09_review_and_qc.md) | [QC and three-pane review](en/09_review_and_qc.md) |
| 10 | [转换为 LeRobot v3.0](zh/10_lerobot_v3.md) | [Converting to LeRobot v3.0](en/10_lerobot_v3.md) |
| 11 | [坐标系约定](zh/11_coordinate_systems.md) | [Coordinate conventions](en/11_coordinate_systems.md) |
| 12 | [运行环境与常见坑](zh/12_environment_and_pitfalls.md) | [Environment and pitfalls](en/12_environment_and_pitfalls.md) |

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

## English

An end-to-end guide to the EgoExplorer capture and reconstruction workflow: hardware, wearing, host setup, capture, calibration, then raw data → SMPL-X → unified 30 Hz export → three-pane review → LeRobot v3.0. Everything is taken from the code and configuration of our production (v3) batch, and the whole processing pipeline runs on one machine. The bill of materials and 3D-printed parts are still to be added. Use the table above or the navigation on the left.

```text
Wear     5× VIVE Ultimate Tracker + gloves (MANUS / Wuji) + GoPro HERO13 (on the head, rigidly coupled to the head tracker)
  └─ per donning: record the 01–17 calibration protocol → calibrate-subject → subject_profile.json
Capture  *.tracker.npz / *.npz (gloves) / *.video.npz / *.mp4 / *.session.json
Rebuild  checks + L/R fixes → projection → hand fusion → 30 Hz export → review pages → gallery (one machine)
Output   unified_30hz/<session>.npz (every stream on one absolute 30 Hz grid)
Release  unified_30hz + mp4 + annotations → LeRobot v3.0 (30 Hz, xyzw quaternions, +Y up)
```
