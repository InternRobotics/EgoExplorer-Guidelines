# 6. 原始数据格式

[返回目录](../index.md) · [English](../en/06_raw_data_format.md)

每条录制一个目录 `session_<日期>_<时间>_<任务名>/`，里面五个文件：

| 文件 | 内容 |
|---|---|
| `<session>.mp4` | GoPro 视频，H.264，可变帧率，每帧的显示时间取自抓帧时刻 |
| `<session>.video.npz` | 每帧的抓帧时间和编码统计 |
| `<session>.tracker.npz` | 五个 tracker 的全局位姿 |
| `<session>.npz` | 两只手套的骨架；Wuji 另有触觉 |
| `<session>.session.json` | 设备和时钟信息 |

重建只读 tracker、手套和视频时间戳三个文件，mp4 只在生成校验页和转换 LeRobot 时用到。

所有时间戳以**微秒**为单位。`t_host_us` 是采集主机的系统时钟（`time.time()`），各路信号都以它为准对齐。

## 6.1 tracker：`*.tracker.npz`

键名前缀为 `tracker_0__pose`，`<role>` 是第 1.3 节的五个角色之一。

| 键 | 形状 | 含义 |
|---|---|---|
| `tracker_0__pose__t_host_us` | (T,) int64 | 采集主机时间（Wuji 方案已换算到 Linux 时钟） |
| `tracker_0__pose__t_us`、`__t_dev_us` | (T,) int64 | 设备时间优先的时间戳、设备时间（tracker 没有独立设备时钟，为 −1） |
| `tracker_0__pose__t_source_host_us` | (T,) int64 | 仅 Wuji 方案：Windows 端的原始时间 |
| `tracker_0__pose.<role>.position` | (T, 3) float32 | 位置，单位米，世界坐标系 +Y 向上 |
| `tracker_0__pose.<role>.orientation` | (T, 4) float32 | 四元数，**顺序为 w, x, y, z**；无效帧可能为 NaN |
| `tracker_0__pose.<role>.valid` | (T,) bool | 该帧是否有效 |
| `meta` | 字符串 | JSON，记录角色、帧数等 |

!!! warning "四元数顺序"
    原始 tracker 文件是 **wxyz**；手套骨架、标定文件和发布的 LeRobot 数据是 **xyzw**。读原始文件请用库里的 `raw_trackers.read_raw_tracker_npz()`，它会转换顺序。另外，有些帧四元数长度为 0 却标为有效，少数录制含 NaN，读取时只用 `valid` 为真且数值有限的样本。

## 6.2 手套：`*.npz`

键名格式为 `{设备}__{数据流}__t_us`、`__t_dev_us`、`__t_host_us` 和 `{设备}__{数据流}`。`glove_0`、`glove_1` 是两只手套，**哪只是左手要看 `session.json` 里的 `slot`**，不能默认 `glove_0` 是左手。

| 键 | Wuji | MANUS（现行，21 节点） | MANUS（旧版，25 节点） |
|---|---|---|---|
| `glove_*__hand_skeleton` | (T, 21, 7) | (T, 21, 7)，采集时已从 25 节点去掉 4 个掌骨节点 | (T, 25, 7) |
| `glove_*__tactile` | (T, 744) | — | — |
| `glove_*__hand_joint_angles.angles`、`.conf` | — | (T, 20)、(T, 5) | 有 |
| 速率 | 约 120 Hz | 骨架约 220 Hz，关节角约 200 Hz | 约 22 Hz |

- `hand_skeleton` 每一行是 `[px, py, pz, qx, qy, qz, qw]`，即位置加 **xyzw** 四元数，节点 0 是手腕。
- 21 节点采用 MediaPipe 的排列，父节点为 `[-1,0,1,2,3,0,5,6,7,0,9,10,11,0,13,14,15,0,17,18,19]`。旧版 25 节点比它多 4 个非拇指的掌骨节点。按节点数判断格式，不要按设备类型判断。
- Wuji 骨架的节点 0 旋转始终是单位四元数，不含手的整体朝向，手的朝向只能来自手腕 tracker。MANUS 的节点 0 含有朝向。
- Wuji 触觉：744 = 24 × 31，按行展开；`-1` 表示无效触点，约占 28%。触觉原始值有基线，未接触时也不为零。
- `meta` 是 JSON，记录 `device_type`（`wuji_glove` 或 `manus_glove`）、每只手套的序列号 `sn` 和 `slot`、各路帧数和时长。

## 6.3 视频时间戳：`*.video.npz`

| 键 | 含义 |
|---|---|
| `t_host_us` | (N,) 每帧的抓帧时间 |
| `fps`、`width`、`height` | 名义值。实际帧率要用 `t_host_us` 计算：Wuji 方案名义 60，实测约 30 |
| `timestamp_mode` | `capture-vfr` |
| `captured_frames`、`queue_dropped_frames`、`read_failures`、`writer_failures` | 采集统计，除帧数外都应为 0 |

## 6.4 设备信息：`*.session.json`

主要字段：`base`（录制名）、`task_name`、`glove_type`（`wuji` 或 `manus`）、`clock`（时钟说明）、`video`、`gloves`（每只手套的 `device_name`、`sn`、`slot`）、`tracker`（角色、传输方式、源时钟和主机时钟、丢帧数、写盘上限）、各路的 `t_host_us_range`，以及 `overlap_s`（各路信号重叠的时长）。

左右修正（第 9.2 节）只改修正副本里的这个文件，原始文件保持不变。

## 6.5 实测采样率

| 方案 | 视频 | 手套 | tracker |
|---|---|---|---|
| MANUS（早期录制） | 52–60 Hz | 约 22 Hz（25 节点）/ 约 220 Hz（21 节点） | **约 15 Hz** |
| MANUS（现行采集程序） | 约 60 Hz | 约 220 Hz | 默认 90 Hz |
| Wuji | **约 30 Hz**（名义 60） | 约 120 Hz | 约 65 Hz |

同一条录制里各路信号的速率不同，不同方案之间 tracker 速率可以差到 4 倍以上。处理时一律按时间戳重采样（位置用线性插值，旋转用 Slerp），不能按数组下标对齐，也不能假设固定帧率。
