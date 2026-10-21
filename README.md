# EgoExplorer Guidelines

**中文** | [English](#english)

EgoExplorer 采集与处理指南：从硬件、佩戴、主机设置、采集和标定，到原始数据 → SMPL-X → 30 Hz 统一导出 → 三联校验 → LeRobot v3.0。内容依据实际使用的代码和参数整理，整条处理管线在一台机器上就能完成。

代码：[InternRobotics/egoexplorer-toolkit](https://github.com/InternRobotics/egoexplorer-toolkit) · 网页版：https://internrobotics.github.io/EgoExplorer-Guidelines/

| # | 中文 | English |
|---|---|---|
| 0 | [总览](docs/zh/00_overview.md) | [Overview](docs/en/00_overview.md) |
| 1 | [硬件与佩戴](docs/zh/01_hardware_and_wearing.md) | [Hardware and wearing](docs/en/01_hardware_and_wearing.md) |
| 2 | [采集主机设置](docs/zh/02_host_setup.md) | [Capture host setup](docs/en/02_host_setup.md) |
| 3 | [采集程序](docs/zh/03_capture_scripts.md) | [Capture program](docs/en/03_capture_scripts.md) |
| 4 | [采集流程](docs/zh/04_data_collection.md) | [Capture workflow](docs/en/04_data_collection.md) |
| 5 | [标定](docs/zh/05_calibration.md) | [Calibration](docs/en/05_calibration.md) |
| 6 | [原始数据格式](docs/zh/06_raw_data_format.md) | [Raw data format](docs/en/06_raw_data_format.md) |
| 7 | [从原始数据到 SMPL-X](docs/zh/07_smplx_reconstruction.md) | [From raw data to SMPL-X](docs/en/07_smplx_reconstruction.md) |
| 8 | [30 Hz 统一导出](docs/zh/08_unified_30hz.md) | [Unified 30 Hz export](docs/en/08_unified_30hz.md) |
| 9 | [质检与三联校验](docs/zh/09_review_and_qc.md) | [QC and three-pane review](docs/en/09_review_and_qc.md) |
| 10 | [转换为 LeRobot v3.0](docs/zh/10_lerobot_v3.md) | [Converting to LeRobot v3.0](docs/en/10_lerobot_v3.md) |
| 11 | [坐标系约定](docs/zh/11_coordinate_systems.md) | [Coordinate conventions](docs/en/11_coordinate_systems.md) |
| 12 | [运行环境与常见问题](docs/zh/12_environment_and_pitfalls.md) | [Environment and common problems](docs/en/12_environment_and_pitfalls.md) |

## 速查

```text
佩戴  5 个 VIVE Ultimate Tracker + 手套（MANUS 或 Wuji）+ GoPro HERO13（头部，与头部 tracker 刚性相连）
标定  每次佩戴录一段 01–17 标定动作 → calibrate-subject → subject_profile.json
采集  每条录制 5 个文件：mp4 / video.npz / npz / tracker.npz / session.json
处理  左右检查与修正 → 投影 → 手部融合 → 30 Hz 导出 → 三联校验页与总览页（scripts/run_pipeline.sh all）
发布  LeRobot v3.0：30 Hz，四元数 xyzw，+Y 向上（scripts/build_lerobot_dataset.py）
```

---

<a id="english"></a>

## English

A guide to capturing and processing EgoExplorer data: hardware, wearing, host setup, capture and calibration, then raw data → SMPL-X → unified 30 Hz export → three-pane review → LeRobot v3.0. Everything follows the code and parameters we actually use, and the whole processing pipeline runs on one machine. Code: [InternRobotics/egoexplorer-toolkit](https://github.com/InternRobotics/egoexplorer-toolkit). Use the table above or the navigation bar.

```text
Wear      5 VIVE Ultimate Trackers + gloves (MANUS or Wuji) + GoPro HERO13 (on the head, rigid to the head tracker)
Calibrate per donning: record the 01–17 protocol → calibrate-subject → subject_profile.json
Capture   5 files per recording: mp4 / video.npz / npz / tracker.npz / session.json
Process   L/R checks and fixes → projection → hand fusion → 30 Hz export → review pages and gallery (scripts/run_pipeline.sh all)
Release   LeRobot v3.0: 30 Hz, xyzw quaternions, +Y up (scripts/build_lerobot_dataset.py)
```
