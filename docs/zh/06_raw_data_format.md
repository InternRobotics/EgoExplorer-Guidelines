# 6. 原始数据格式

[返回目录](../index.md) · [English](../en/06_raw_data_format.md)

每条 session 一个目录 `session_<日期>_<时间>_<任务>/`，五个文件：

```text
<session>.mp4            GoPro 视频（H.264，按抓帧时刻写 PTS，VFR）
<session>.video.npz      每帧主机时间戳与编码统计
<session>.tracker.npz    五个 tracker 的全局 6-DoF 位姿
<session>.npz            两只手套（骨架；Wuji 另有触觉）
<session>.session.json   设备与时钟元信息
```

重建只读 tracker、手套和视频时间戳；mp4 只在生成审阅页和转换 LeRobot 时用到。

所有时间戳单位为**微秒**；`t_host_us` 是主机墙钟（`time.time()`），是统一的主时间轴。

## 6.1 `*.tracker.npz`

键前缀固定为 `tracker_0__pose`：

| 键 | 形状 | 说明 |
|---|---|---|
| `tracker_0__pose__t_host_us` | (T,) int64 | 主机墙钟（Wuji 方案已换算到 Linux 墙钟） |
| `tracker_0__pose__t_us` / `__t_dev_us` | (T,) int64 | 设备优先时间 / 设备时间 |
| `tracker_0__pose__t_source_host_us` | (T,) int64 | 仅 Wuji 方案：Windows 原始墙钟 |
| `tracker_0__pose.<role>.position` | (T, 3) float32 | 米，tracker 世界坐标（+Y 向上） |
| `tracker_0__pose.<role>.orientation` | (T, 4) float32 | **四元数 `w, x, y, z` 顺序**；无效帧可能为 NaN |
| `tracker_0__pose.<role>.valid` | (T,) bool | 该帧是否有效 |
| `meta` | 标量字符串 | JSON：roles、frames 等 |

`<role>` ∈ `camera, left_wrist, right_wrist, left_ankle, right_ankle`。

> ⚠️ 原始文件的四元数是 **wxyz**；发布的 LeRobot 数据和手套骨架是 **xyzw**。读原始文件请用库里的 `raw_trackers.read_raw_tracker_npz()`，它会重排顺序。另外：部分帧四元数是零范数却仍标 valid，部分 session 有 NaN，读取时要按 `valid & 有限值` 过滤。

## 6.2 `*.npz`（手套）

键格式统一为 `{device}__{stream}__t_us / __t_dev_us / __t_host_us` 和 `{device}__{stream}`，`glove_0` / `glove_1` 是两只手，**哪只是左手看 `session.json` 的 `slot`**（不要默认 glove_0 = 左）。

| 键 | Wuji | MANUS（新，21 点） | MANUS（旧，25 点） |
|---|---|---|---|
| `glove_*__hand_skeleton` | (T, 21, 7) | (T, 21, 7)（采集端已从 25 点降为 21 点） | (T, 25, 7) |
| `glove_*__tactile` | (T, 744) | — | — |
| `glove_*__hand_joint_angles.angles` / `.conf` | — | (T, 20) / (T, 5) | 有 |
| 速率 | ~120 Hz | 骨架 ~220 Hz，角度 ~200 Hz | ~22 Hz |

- `hand_skeleton` 每行是 `[px, py, pz, qx, qy, qz, qw]`（位置 + **xyzw** 四元数），节点 0 是手腕。
- **21 点布局 = MediaPipe**，父节点 `[-1,0,1,2,3,0,5,6,7,0,9,10,11,0,13,14,15,0,17,18,19]`；25 点 MANUS 布局多了 4 个非拇指掌骨节点。加载器**按节点数**判断布局，不要按设备类型（新 MANUS 也是 21 点）。
- Wuji 的节点 0 四元数恒为单位阵，**不带手部全局朝向**，手的朝向只能从腕部 tracker 来；MANUS 的节点 0 带世界朝向。
- Wuji 触觉：744 = 24×31 矩阵，行优先展开；`-1` 表示无效（未接线）taxel，约 28%。触觉原始值有基线，不为零。
- `meta`：JSON，含 `device_type`（`wuji_glove` / `manus_glove`）、各手 `sn` 与 `slot`、每路帧数与时长、`host_minus_dev_us`。

## 6.3 `*.video.npz`

| 键 | 说明 |
|---|---|
| `t_host_us` | (N,) 每帧抓取时刻的主机墙钟 |
| `fps` / `width` / `height` | **名义值**；实际帧率请用 `t_host_us` 算（Wuji 方案名义 60、实测 ~30） |
| `timestamp_mode` | `capture-vfr` |
| `captured_frames`, `queue_dropped_frames`, `read_failures`, `writer_failures` | 采集统计，正常应全为 0（除帧数） |

## 6.4 `*.session.json`

关键字段：`base`、`task_name`、`glove_type`（`wuji`/`manus`）、`clock`（时钟说明）、`video{…}`、`gloves[{device_name, sn, slot}]`、`tracker{roles, transport, source_clock, host_clock, dropped_frames, record_fps_limit}`、各流的 `t_host_us_range`、`overlap_s`（所有流重叠的时长）。

左右修正（第 9 章）只改写修正副本里的这个文件（交换 `slot`）并记录证据，原始文件不动。

## 6.5 实测采样率（写代码前必须知道）

| 方案 | 视频 | 手套 | tracker |
|---|---|---|---|
| MANUS（旧批次） | ~52–60 Hz | ~22 Hz（25 点）/ ~220 Hz（21 点） | **~15 Hz**（采集端修复前） |
| MANUS（现行采集代码） | ~60 Hz | ~220 Hz | 主机时钟采样，默认 90 Hz |
| Wuji | **~30 Hz**（名义 60） | ~120 Hz | ~65 Hz |

同一份录制里三个流三个速率，跨方案 tracker 差 4 倍。任何阶段都按时间戳重采样（`np.interp` / Slerp），**不要按索引对齐，不要假设固定帧率**。
