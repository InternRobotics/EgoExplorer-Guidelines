# EgoExplorer Guidelines

**中文** | [English](#english)

EgoExplorer 采集与重建全流程指南：从硬件、穿戴、主机设置、采集、标定，到原始数据 → SMPL-X → 30 Hz 统一导出 → 三联校验 → LeRobot v3.0。内容依据我们在 H 集群（以及后来的 T 集群）上实际跑过的代码和配置整理，所有参数都是生产批次（v3）真正在用的取值。

> 状态：内部草稿（私有仓库）。采集脚本一章暂时留空，待补充后再随工具包一起公开。

## 目录

| # | 中文 | English |
|---|---|---|
| 0 | [总览与数据流](docs/zh/00_overview.md) | [Overview and data flow](docs/en/00_overview.md) |
| 1 | [硬件组成与穿戴](docs/zh/01_hardware_and_wearing.md) | [Hardware and wearing](docs/en/01_hardware_and_wearing.md) |
| 2 | [采集主机设置](docs/zh/02_host_setup.md) | [Capture host setup](docs/en/02_host_setup.md) |
| 3 | [采集脚本（待补充）](docs/zh/03_capture_scripts.md) | [Capture scripts (TBD)](docs/en/03_capture_scripts.md) |
| 4 | [如何采集数据](docs/zh/04_data_collection.md) | [Collecting data](docs/en/04_data_collection.md) |
| 5 | [标定与标定数据采集](docs/zh/05_calibration.md) | [Calibration and calibration recordings](docs/en/05_calibration.md) |
| 6 | [原始数据格式](docs/zh/06_raw_data_format.md) | [Raw data format](docs/en/06_raw_data_format.md) |
| 7 | [原始数据 → SMPL-X（含手部融合、重训练）](docs/zh/07_smplx_reconstruction.md) | [Raw → SMPL-X (hands, retraining)](docs/en/07_smplx_reconstruction.md) |
| 8 | [30 Hz 统一导出](docs/zh/08_unified_30hz.md) | [Unified 30 Hz export](docs/en/08_unified_30hz.md) |
| 9 | [质检与三联校验](docs/zh/09_review_and_qc.md) | [QC and three-pane review](docs/en/09_review_and_qc.md) |
| 10 | [转换为 LeRobot v3.0](docs/zh/10_lerobot_v3.md) | [Converting to LeRobot v3.0](docs/en/10_lerobot_v3.md) |
| 11 | [坐标系约定](docs/zh/11_coordinate_systems.md) | [Coordinate conventions](docs/en/11_coordinate_systems.md) |
| 12 | [集群运行与常见坑](docs/zh/12_cluster_and_pitfalls.md) | [Cluster operations and pitfalls](docs/en/12_cluster_and_pitfalls.md) |

## 一页速查

```text
穿戴 5×VIVE Ultimate Tracker + 手套(MANUS / Wuji) + GoPro HERO13(头部，与头部 tracker 刚性相连)
  └─ 每次穿戴：采一段 01–17 标定动作 → calibrate-subject → subject_profile.json
采集  *.tracker.npz / *.npz(手套) / *.video.npz / *.mp4 / *.session.json   → rclone 上传到桶
重建  run_batch_v3.sh view → project → check → fix-lr → project(重跑) → fuse → export → videos → review → gallery
输出  unified_30hz/<session>.npz（所有流在同一条 30 Hz 绝对时间栅格上）
发布  unified_30hz + mp4 + 标注 → LeRobot v3.0（30 Hz，xyzw 四元数，+Y 向上）
```

---

<a id="english"></a>

# EgoExplorer Guidelines (English)

An end-to-end guide to the EgoExplorer capture and reconstruction workflow: hardware, wearing, host setup, capture, calibration, then raw data → SMPL-X → unified 30 Hz export → three-pane review → LeRobot v3.0. Everything here is taken from the code and configuration we actually ran on the H cluster (and later the T cluster); parameter values are the ones used by the production (v3) batch.

> Status: internal draft (private repository). The capture-script chapter is a placeholder for now and will be filled in before the toolkit goes public.

See the table of contents above; the English chapters live under [`docs/en/`](docs/en/).

## One-page cheat sheet

```text
Wear     5× VIVE Ultimate Tracker + gloves (MANUS / Wuji) + GoPro HERO13 (on the head, rigidly coupled to the head tracker)
  └─ per donning: record the 01–17 calibration protocol → calibrate-subject → subject_profile.json
Capture  *.tracker.npz / *.npz (gloves) / *.video.npz / *.mp4 / *.session.json   → rclone upload to the bucket
Rebuild  run_batch_v3.sh view → project → check → fix-lr → project (rerun) → fuse → export → videos → review → gallery
Output   unified_30hz/<session>.npz (every stream on one absolute 30 Hz grid)
Release  unified_30hz + mp4 + annotations → LeRobot v3.0 (30 Hz, xyzw quaternions, +Y up)
```
