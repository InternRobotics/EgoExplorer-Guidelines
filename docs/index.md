# EgoExplorer Guidelines

**中文** | [English](#english)

EgoExplorer 采集与重建全流程指南：从硬件、穿戴、主机设置、采集、标定，到原始数据 → SMPL-X → 30 Hz 统一导出 → 三联校验 → LeRobot v3.0。内容依据我们在 H 集群（以及后来的 T 集群）上实际跑过的代码和配置整理，参数都是生产批次（v3）真正在用的取值。

> 状态：草稿。采集脚本一章暂时留空，待补充。

| # | 中文 | English |
|---|---|---|
| 0 | [总览与数据流](zh/00_overview.md) | [Overview and data flow](en/00_overview.md) |
| 1 | [硬件组成与穿戴](zh/01_hardware_and_wearing.md) | [Hardware and wearing](en/01_hardware_and_wearing.md) |
| 2 | [采集主机设置](zh/02_host_setup.md) | [Capture host setup](en/02_host_setup.md) |
| 3 | [采集脚本（待补充）](zh/03_capture_scripts.md) | [Capture scripts (TBD)](en/03_capture_scripts.md) |
| 4 | [如何采集数据](zh/04_data_collection.md) | [Collecting data](en/04_data_collection.md) |
| 5 | [标定与标定数据采集](zh/05_calibration.md) | [Calibration and calibration recordings](en/05_calibration.md) |
| 6 | [原始数据格式](zh/06_raw_data_format.md) | [Raw data format](en/06_raw_data_format.md) |
| 7 | [原始数据 → SMPL-X（含手部融合、重训练）](zh/07_smplx_reconstruction.md) | [Raw → SMPL-X (hands, retraining)](en/07_smplx_reconstruction.md) |
| 8 | [30 Hz 统一导出](zh/08_unified_30hz.md) | [Unified 30 Hz export](en/08_unified_30hz.md) |
| 9 | [质检与三联校验](zh/09_review_and_qc.md) | [QC and three-pane review](en/09_review_and_qc.md) |
| 10 | [转换为 LeRobot v3.0](zh/10_lerobot_v3.md) | [Converting to LeRobot v3.0](en/10_lerobot_v3.md) |
| 11 | [坐标系约定](zh/11_coordinate_systems.md) | [Coordinate conventions](en/11_coordinate_systems.md) |
| 12 | [集群运行与常见坑](zh/12_cluster_and_pitfalls.md) | [Cluster operations and pitfalls](en/12_cluster_and_pitfalls.md) |

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

## English

An end-to-end guide to the EgoExplorer capture and reconstruction workflow: hardware, wearing, host setup, capture, calibration, then raw data → SMPL-X → unified 30 Hz export → three-pane review → LeRobot v3.0. Everything is taken from the code and configuration we actually ran on the H cluster (and later the T cluster); parameter values are the ones used by the production (v3) batch. The capture-script chapter is a placeholder for now. Use the table above or the navigation on the left.

```text
Wear     5× VIVE Ultimate Tracker + gloves (MANUS / Wuji) + GoPro HERO13 (on the head, rigidly coupled to the head tracker)
  └─ per donning: record the 01–17 calibration protocol → calibrate-subject → subject_profile.json
Capture  *.tracker.npz / *.npz (gloves) / *.video.npz / *.mp4 / *.session.json   → rclone upload to the bucket
Rebuild  run_batch_v3.sh view → project → check → fix-lr → project (rerun) → fuse → export → videos → review → gallery
Output   unified_30hz/<session>.npz (every stream on one absolute 30 Hz grid)
Release  unified_30hz + mp4 + annotations → LeRobot v3.0 (30 Hz, xyzw quaternions, +Y up)
```
