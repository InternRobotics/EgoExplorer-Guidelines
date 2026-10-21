# 2. 采集主机设置

[返回目录](../index.md) · [English](../en/02_host_setup.md)

采集代码在 [egoexplorer-toolkit](https://github.com/InternRobotics/egoexplorer-toolkit) 的 `capture/` 目录，一套代码支持两种手套，入口分别是 `record_ego_wuji.py` 和 `record_ego_manus.py`。

## 2.1 两种方案

=== "Wuji 方案"

    ```mermaid
    flowchart LR
        W[Wuji 手套 ×2] -- 以太网 --> L
        G[GoPro + Media Mod] -- HDMI → 采集卡 --> L
        T[VIVE Ultimate Tracker ×5] --> WIN[Windows：SteamVR<br/>tracker_sender_windows.py]
        WIN -- TCP 8766 --> L[Linux 采集机<br/>record_ego_wuji.py]
        L --> F[session 目录]
    ```

    Linux 采集机负责全部录制：视频和手套直接按 Linux 系统时钟打时间戳。tracker 由 Windows 主机读取，每帧连同 Windows 时钟一起发给 Linux。两端每 2 秒对一次时，Linux 用往返延迟最小的那次采样估计两台机器的时钟差，把 tracker 时间换算成 Linux 时间。对时完成前，以及时钟差超过 5 秒（视为异常）时，改用 Linux 收到数据的时刻。实测一段录制内的时钟漂移中位数为 4.2 ms。

=== "MANUS 方案"

    ```mermaid
    flowchart LR
        M[MANUS 手套] --> MC[MANUS Core<br/>SDKMinimalClient_Windows.exe] -- UDP 9000 --> R
        T[VIVE Ultimate Tracker ×5] -- SteamVR（本机） --> R
        G[GoPro + Media Mod] -- HDMI → 采集卡 --> R[Windows 采集机<br/>record_ego_manus.py]
        R --> F[session 目录]
    ```

    一台 Windows 主机完成全部采集，三路数据用同一个系统时钟打时间戳，写盘时就已对齐。MANUS 发来的是 25 节点手部骨架，采集程序去掉 4 个非拇指的掌骨节点，按 21 节点（与 Wuji、MediaPipe 相同）保存。

## 2.2 SteamVR 与 tracker（两种方案相同）

1. 安装 **VIVE Hub** 和 **SteamVR**，把 5 个 tracker 配对到无线接收器，按 VIVE Hub 的引导建好场地地图。
2. 在 SteamVR 中把 SteamVR 设为当前的 **OpenXR Runtime**。
3. 按第 1.3 节给每个 tracker 分配角色。角色为 Handheld object 的 tracker 不会输出位姿。发送端启动时会打印识别到的序列号和角色，可以据此核对。

tracker 的采样率：

| 方案 | 做法 | 默认值 |
|---|---|---|
| Wuji | Windows 端发送有上限（`--send-fps`），Linux 端写盘也有上限（`--tracker-record-fps`）。部分 OpenXR 运行时不限速时会产生近千帧每秒的重复位姿，挤占手套数据的接收 | 发送 90，写盘 120 |
| MANUS | 按系统时钟直接查询 tracker 位姿（`--tracker-sample-fps`），不再跟随 SteamVR 约 15 Hz 的后台刷新节奏 | 90 |

!!! note "说明"
    修复采样方式之前录的 MANUS 数据，tracker 只有约 15 Hz（第 6.5 节）。

## 2.3 Wuji 方案的 Linux 采集机

推荐 Ubuntu 22.04 或更新版本（Wuji SDK 的要求），Python 3.10 或 3.11。

```bash
cd capture
python3 -m venv .venv && source .venv/bin/activate
python -m pip install -r requirements.txt
```

### 连接手套

每只手套接一个网口（一般用 USB 网卡），出厂地址为左手 `192.168.1.100`、右手 `192.168.1.101`。配置脚本会找到对应网口，写入持久的网络配置，并实际扫描、连接一次做验证：

```bash
# 自动配置并验证
./scripts/setup_wuji_gloves.sh
# 只检查，不修改
./scripts/setup_wuji_gloves.sh --check
# 自动识别不出来时，手动指定网口
./scripts/setup_wuji_gloves.sh --interface enx001 --interface enx002
# 要求两只手套都在线；开着 UFW 时放行 SDK 端口
./scripts/setup_wuji_gloves.sh --expected-gloves 2 --allow-ufw
```

脚本不会清除已有的网卡地址，也不会改动承载默认路由的网口。手套 IP 改过的话，用 `--left-ip`、`--right-ip`、`--host-prefix` 指定。

### 手套标定

每副手套都要先做手部标定，SDK 才能得到这副手套和佩戴者的手部模型。标定结果存在 Wuji SDK 的某个"用户"下。采集程序启动时导入 `wuji_calibration/calibration_bundle.zip`，并切换到这个文件所属的用户（`--wuji-calibration` 指定其它路径，`--no-wuji-calibration` 跳过导入、直接用 SDK 的当前用户）。这个文件不放进代码仓库，按下面的步骤生成（需要 `wuji-sdk>=2026.7.21`）：

1. 关闭 Wuji Studio（它会占用 SDK），用 `setup_wuji_gloves.sh --check` 确认两只手套都能扫到。
2. 新建一个 SDK 用户，用来存放这副手套的标定，记下返回的 `user_id`：
    ```bash
    python scripts/wuji_calibration_tool.py users                 # 列出已有用户，* 为当前用户
    python scripts/wuji_calibration_tool.py create-user "rig-A"
    ```
3. 标定，两种方式任选一种：
    - **Wuji Studio（推荐）**：打开 Wuji Studio，选中上一步建的用户，按 [Wuji Studio 文档](https://docs.wuji.tech/docs/en/wuji-studio/latest/) 逐只手套完成手部标定，完成后关闭 Wuji Studio；
    - **脚本**：`python scripts/wuji_calibration_tool.py calibrate --user <user_id>`，终端会逐步提示要摆的手势。
4. 导出并校验：
    ```bash
    python scripts/wuji_calibration_tool.py export --user <user_id>   # 写到 wuji_calibration/calibration_bundle.zip
    python scripts/wuji_calibration_tool.py preview                   # 只校验，不导入
    ```
5. 把 `calibration_bundle.zip` 拷到其它采集机的 `capture/wuji_calibration/` 下。采集程序启动时自动导入，终端打印"已加载标定用户 …"即为成功。也可以用 `wuji_calibration_tool.py import` 手动导入。

换手套或手套返修之后，重新标定并导出。

### 采集卡

```bash
sudo apt install v4l-utils
v4l2-ctl --list-devices                        # 找到采集卡的 /dev/videoN
v4l2-ctl -d /dev/videoN --list-formats-ext     # 查看支持的分辨率和帧率
```

视频的抓帧和编码在不同线程里进行。MP4 里每一帧的显示时间取自抓帧时刻，中途丢帧不会让视频变快。

### 网络

Windows 发送端要能访问 Linux 的 TCP 8766 端口：

```bash
hostname -I                                                  # Linux 的局域网 IP
sudo ufw allow from <WINDOWS_IP> to any port 8766 proto tcp  # 开着 UFW 时
```

不要把 8766 端口暴露到公网。在局域网内也建议给两端设相同的 `--tracker-token`（或环境变量 `EGO_TRACKER_TOKEN`）。

## 2.4 MANUS 方案的 Windows 采集机

MANUS 手套的数据要经过两个程序才到达采集程序：

- **MANUS Core**：MANUS 官方的手套服务，负责连接手套、做手部标定、解算手指姿态。从 [MANUS 资源页](https://docs.manus-meta.com/3.0.0/Resources/) 下载安装，按官方说明完成配对和标定。
- **UDP 发送端**（`SDKMinimalClient_Windows.exe`）：基于 MANUS SDK 编译，从 MANUS Core 读取手套数据，再通过 UDP 发给 `record_ego_manus.py`。它和 `ManusSDK.dll` 受 MANUS SDK 许可约束，不放在仓库里，需要自行获取并放到 `capture/ManusClient/`。获取方式和 UDP 数据包格式见该目录下的 `README.md`。

依赖与 Linux 端相同（`requirements.txt`），在 Windows 上会安装 `pyopenxr`、不装 `wuji-sdk`。采集卡在 Windows 上通过 DirectShow 读取。

启动顺序：MANUS Core（确认手套在线）→ UDP 发送端 → 采集程序。

## 2.5 采集前检查

=== "Wuji 方案"

    - [ ] SteamVR 里五个 tracker 都在线，角色正确
    - [ ] `setup_wuji_gloves.sh --check` 通过，两只手套都能扫到，Wuji Studio 已关闭
    - [ ] `wuji_calibration/calibration_bundle.zip` 对应当前这副手套
    - [ ] 采集页面显示 tracker 已连接，相机画面和 tracker 位姿正常
    - [ ] 磁盘空间充足

=== "MANUS 方案"

    - [ ] SteamVR 里五个 tracker 都在线，角色正确
    - [ ] MANUS Core 里手套在线，UDP 发送端已启动且目标 IP、端口正确
    - [ ] 采集页面显示 tracker 已连接，相机画面和 tracker 位姿正常
    - [ ] 磁盘空间充足
