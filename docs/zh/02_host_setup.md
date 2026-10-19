# 2. 采集主机设置

[返回目录](../index.md) · [English](../en/02_host_setup.md)

采集端代码分两层：`devices/`（各设备驱动）和 `record_ego/`（采集主程序 + 网页控制台）。录制端有两种拓扑，按手套类型区分。

## 2.1 两种拓扑

### MANUS 方案：单台 Windows 主机

```text
VIVE Ultimate Tracker ×5 ──dongle──► Windows：VIVE Hub + SteamVR（headless）──OpenXR──► 采集程序（本机）
MANUS 手套 ──► Windows：MANUS Core + 改版 SDKMinimalClient ──UDP:9000──► 采集程序（本机）
GoPro ──HDMI──► 采集卡 ──USB──► 采集程序（本机，DirectShow）
```

三路数据都在**同一台机器、同一个墙钟**（`time.time()`）上打时间戳，落盘即对齐。

### Wuji 方案：Linux 采集机 + Windows tracker 机

```text
VIVE Ultimate Tracker ×5 ──► Windows：VIVE Hub + SteamVR + tracker 发送端 ──TCP:8766──► Linux 采集程序
Wuji 手套 ×2 ──USB/SDK──► Linux：wuji_sdk ──► Linux 采集程序
GoPro ──HDMI──► 采集卡 ──USB──► Linux 采集程序（V4L2）
```

视频和手套用 Linux 墙钟；tracker 的时间戳先记 Windows 墙钟，接收端用 NTP 式往返采样估计两台机器的时钟差，换算到 Linux 墙钟（每个 session 重新估计）。实测 session 内总漂移中位 4.2 ms、拟合抖动 0.39 ms，质量足够。

## 2.2 Windows 端（tracker）

1. 安装 **VIVE Hub** 和 **SteamVR**，把 5 个 Ultimate Tracker 配对到 dongle 上，按 VIVE Hub 的流程建好场地地图（inside-out 定位需要）。
2. 在 SteamVR 的 tracker 管理里给每个 tracker 分配角色：`camera`、`left_wrist`、`right_wrist`、`left_ankle`、`right_ankle`。
   - 角色必须在驱动支持的列表里，否则这个 tracker 永远不会出现（`devices/vive_tracker.py` 的 `ROLES_FULL` / `ROLES_CLASSIC`）。
   - **角色为 "Handheld object" 的 tracker 不会被上报。**
3. SteamVR 以 headless 方式运行；驱动 `devices/vive_tracker.py` 基于 pyopenxr，用 Win32 performance-counter 扩展换算 XrTime。
4. Wuji 方案下，在 Windows 上运行 tracker 发送端，连到 Linux 采集机的 TCP 8766 端口（可用 `--tracker-token` / 环境变量 `EGO_TRACKER_TOKEN` 设共享密钥）。

> ⚠️ 已知问题：MANUS 方案下 tracker 实际录制帧率只有 ~15 Hz。原因是 OpenXR 会话以 `MND_HEADLESS` 建立、每帧提交零图层，运行时把帧循环降到后台节奏（90/6 ≈ 15 Hz），而录制帧率等于 `xr.wait_frame()` 的返回频率。根本修法是把位姿采样从 XR 帧循环里拿出来，开一个高频线程直接 `xrLocateSpace`。Wuji 方案走 Windows 发送端，帧率约 65 Hz，没有这个问题。

## 2.3 Linux 端（Wuji 方案的采集机）

1. **采集卡**：确认设备号与支持的格式
   ```bash
   sudo apt install v4l-utils
   v4l2-ctl --list-devices
   v4l2-ctl -d /dev/videoN --list-formats-ext
   ```
2. **编码**：录制用系统 ffmpeg 的 libx264（`--video-codec h264 --crf 20 --h264-preset ultrafast`）。opencv-python 自带的 ffmpeg 往往没有 libx264，会静默退回 mp4v，所以要装系统 ffmpeg。视频按逐帧抓取时刻写 PTS（`timestamp_mode = capture-vfr`），编码快慢不影响时间轴。
3. **Wuji SDK**：安装 `wuji_sdk`；采集程序启动时导入标定 bundle（默认 `wuji_calibration/calibration_bundle.zip`），扫描并连接全部手套，订阅 `tactile` + `hand_skeleton` 两路（为保证视频实时抓帧，默认不订阅派生分区、关节角、EMF 和点云）。
4. **网络**：Windows 发送端能访问 Linux 的 TCP 8766。
5. **上传**：配置 rclone 远端（例如 `eailab-hdd2:`），采集程序可用 `--rclone-remote` 手动或按容量阈值自动上传（见第 4 章）。

## 2.4 MANUS 方案的 Windows 采集机

1. 安装 MANUS Core，运行改版的 `SDKMinimalClient`，把原始骨骼帧以 UDP 发到本机 9000 端口（`--manus-ip 0.0.0.0 --manus-port 9000`）。
2. MANUS 的 UDP 帧只有自增帧号，没有设备时钟；时间戳取收到包时的主机墙钟，所以与视频、tracker 天然同钟。
3. 采集卡在 Windows 上走 DirectShow（`CAP_DSHOW`）。

## 2.5 时钟与同步要点

- 所有时间戳单位都是**微秒**，主时间轴是**主机墙钟** `t_host_us`。
- 手套额外存设备时钟 `t_dev_us`（MANUS 没有，记 −1）；Wuji 方案的 tracker 额外存 Windows 原始时间 `t_source_host_us`，`t_host_us − t_source_host_us` 就是该帧用的时钟换算量。
- 三个流三个速率，下游一律按时间戳重采样，**不要按索引对齐**（第 8、12 章）。

## 2.6 采集前自检清单

- [ ] 五个 tracker 在 SteamVR 里角色正确、全部在线，无跳帧
- [ ] 两只手套都在线，左右 slot 与实际佩戴一致
- [ ] 采集页面有视频画面，分辨率和帧率与预期一致
- [ ] 磁盘剩余空间充足（默认容量上限 50 GB，超出后自动上传清理）
- [ ] Wuji 方案：页面显示 Windows 发送端已连接
