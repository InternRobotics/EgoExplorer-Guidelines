# 3. 采集程序

[返回目录](../index.md) · [English](../en/03_capture_scripts.md)

## 3.1 代码结构

```text
capture/
├── record_ego_wuji.py              Wuji 方案入口（Linux）
├── record_ego_manus.py             MANUS 方案入口（Windows）
├── record_ego_app.py               设备连接、网页服务与录制流程
├── record_ego_page.html            控制页面
├── tracker_sender_windows.py       Windows 端 tracker 发送程序（Wuji 方案）
├── visualize_session.py            回看一条录制
├── record_ego/                     配置、录制控制、写盘、上传
├── devices/                        各设备驱动
├── network_bridge/                 tracker 数据的网络协议
├── ManusClient/                    MANUS UDP 发送端（需自行获取，见目录内 README）
├── wuji_calibration/               Wuji 标定文件的导入与导出
├── scripts/
│   ├── setup_wuji_gloves.sh        Wuji 手套网络配置与检查
│   ├── wuji_calibration_tool.py    Wuji 手套标定与标定文件导出
│   ├── start_tracker_sender.ps1    启动 Windows 端 tracker 发送程序
│   ├── wuji_launch.sh              Wuji 方案一键启动（按本机修改）
│   ├── rclone_linux.sh             上传数据（Linux）
│   └── rclone_win.bat              上传数据（Windows）
└── task_setting/example.json       任务列表示例
```

## 3.2 任务列表

`--task-setting` 指向一个 JSON 文件，内容是任务名数组。控制页面上按顺序切换任务，任务名会写进录制名和 `session.json`：

```json
["物料称量", "萃取", "开反应", "旋蒸", "烘箱取料", "过柱"]
```

## 3.3 启动

=== "Wuji 方案"

    先在 Linux 上启动采集程序：

    ```bash
    cd capture && source .venv/bin/activate
    python record_ego_wuji.py \
      --task-setting task_setting/example.json \
      --tracker-host 0.0.0.0 --tracker-port 8766 \
      --save-dir ./data
    ```

    也可以按本机路径改好 `scripts/wuji_launch.sh` 后一键启动。

    再在 Windows 的 `capture/` 目录下启动 tracker 发送程序：

    ```powershell
    .\scripts\start_tracker_sender.ps1 -LinuxHost <LINUX_IP> -Port 8766
    # 或
    python tracker_sender_windows.py --host <LINUX_IP> --port 8766
    ```

    - 两端的端口（以及设了的话 token）必须一致。连接断开后 Windows 端每 2 秒重连一次，Linux 端照常录制视频和手套。
    - Windows 端只需允许 Python 访问局域网，不用开放入站端口。
    - 在 Windows 终端里也能控制录制：`r` 开始，`e` 结束并保存，`Esc` 关闭按键监听（位姿照常发送）。开始前的倒计时默认沿用 Linux 端的 `--record-countdown`，可以用 `-Countdown` / `--countdown` 覆盖；不需要按键控制时加 `-NoKeyboard` / `--no-keyboard`。

=== "MANUS 方案"

    按第 2.4 节启动 MANUS Core 和 UDP 发送端，然后：

    ```bash
    python record_ego_manus.py \
      --task-setting task_setting/example.json \
      --manus-ip 0.0.0.0 --manus-port 9000
    ```

    tracker 由本机直接读取。没有 tracker 时加 `--no-tracker`，相机和手套照常采集。

## 3.4 控制页面

控制页面默认在 `http://127.0.0.1:8770/`。要从其它电脑访问，加 `--host 0.0.0.0` 并放行端口。

<div style="display:flex; gap:12px; align-items:flex-start" markdown>

<div style="flex:1.985 1 0; min-width:0" markdown>

![MANUS 方案的控制页面](../assets/img/interface.png)

*MANUS 方案*

</div>

<div style="flex:1.778 1 0; min-width:0" markdown>

![Wuji 方案的控制页面](../assets/img/capture_session_viewer.jpg)

*Wuji 方案*

</div>

</div>

两种方案的页面布局相同：左侧是 GoPro 画面，右上是五个 tracker 的位姿和坐标，右下是左右手骨架（Wuji 方案另有 24×31 触觉热力图），底部是操作栏。Wuji 页面的相机监看画面默认缩到 640×360（`--preview-width`、`--preview-height`），不影响录制分辨率。

键盘操作与终端相同：

| 键 | 作用 |
|---|---|
| `r` | 倒计时（默认 3 秒，可在页面上改）后开始录制 |
| `e` | 结束并保存 |
| `d` | 倒计时中按下取消；录制中按下则终止并丢弃，不写盘 |
| `q` | 退出，未保存的录制会被丢弃 |

## 3.5 常用参数

| 参数 | 默认值 | 说明 |
|---|---|---|
| `--cam-device` | `0` | 采集卡编号或 `/dev/video*` 路径 |
| `--cam-width` `--cam-height` `--cam-fps` | `2560` `1440` `60` | 向采集卡请求的分辨率和帧率，实际值以采集卡输出为准，记录在 `session.json` |
| `--video-codec` `--crf` `--h264-preset` `--h264-threads` | `h264` `20` `ultrafast` `4` | 实时 H.264 编码 |
| `--no-camera` | | 不接相机 |
| `--wuji-calibration` / `--no-wuji-calibration` | `wuji_calibration/calibration_bundle.zip` | Wuji 手套标定文件 |
| `--tracker-host` `--tracker-port` `--tracker-token` | `0.0.0.0` `8766` 空 | Wuji 方案接收 tracker 数据 |
| `--tracker-record-fps` | `120` | Wuji 方案 tracker 写盘上限 |
| `--tracker-sample-fps` | `90` | MANUS 方案 tracker 采样率 |
| `--no-tracker` | | 不接 tracker |
| `--save-dir` | `./data` | 录制保存目录 |
| `--record-countdown` | `3` | 开始前倒计时（秒） |
| `--rclone-remote` `--rclone-config` | 空 | 上传目标（可选） |
| `--storage-limit-gb` `--storage-check-interval` | `50` `30`（分钟） | 数据目录超过上限时自动上传并清理 |
| `--host` `--web-port` | `127.0.0.1` `8770` | 控制页面地址和端口 |

完整参数见 `python record_ego_wuji.py --help` 和 `python record_ego_manus.py --help`。

## 3.6 上传（可选）

录制默认只保存在本机的 `--save-dir` 下。本机空间不够时，可以开通一个兼容 S3 的对象存储（阿里云 OSS、腾讯云 COS、AWS S3 等），用 rclone 上传：

1. 安装 [rclone](https://rclone.org/downloads/)；
2. 运行 `rclone config` 新建一个远端，例如命名为 `myremote`，填入存储类型、Endpoint 和密钥；
3. 用 `rclone lsd myremote:<bucket>` 验证能列出内容。

之后有两种上传方式：

- **在采集程序里上传。** 启动时加 `--rclone-remote myremote:<bucket>/ego_data`（或设环境变量 `EGO_RCLONE_REMOTE`），页面上的"上传数据"按钮随即可用，它执行 `rclone copy`，跳过远端已有的文件。数据目录超过 `--storage-limit-gb`（`0` 表示不检查）时，程序自动执行 `rclone move`，只删除确认已上传的本地文件。rclone 不在 PATH 里时用 `--rclone-bin` 指定，配置文件不在默认位置时用 `--rclone-config` 指定。
- **用脚本上传。** 把 `scripts/rclone_linux.sh` 或 `scripts/rclone_win.bat` 开头的 `LOCAL_PATH`、`REMOTE_PATH` 改成自己的路径后运行。需要过滤文件或上传后删除本地文件时，修改脚本里那条 rclone 命令。

没有配置 rclone 时，"上传数据"按钮不可用，超过存储上限也不会自动清理，需要自己留意磁盘空间。

## 3.7 常见问题

| 现象 | 依次检查 |
|---|---|
| Linux 一直显示等待 tracker | Windows 端 `--host` 填的是 Linux 的局域网 IP（不是 127.0.0.1）；两端端口和 token 一致；`Test-NetConnection <LINUX_IP> -Port 8766` 能通；Linux 防火墙；两台机器在同一网段；Linux 端先启动 |
| Windows 端没有 tracker 位姿 | 依赖已安装；SteamVR 是当前的 OpenXR Runtime；tracker 已配对并分配了角色（不能是 Handheld object） |
| Linux 找不到 Wuji 手套 | 已安装 `wuji-sdk`，没有其它程序占用 SDK；`setup_wuji_gloves.sh --check`；左右手序列号与当前 SDK 用户的标定是否对应 |
