# 3. 采集脚本

[返回目录](../index.md) · [English](../en/03_capture_scripts.md)

## 3.1 代码结构

```text
capture/
├── record_ego_wuji.py              Wuji 方案入口（Linux）
├── record_ego_manus.py             MANUS 方案入口（Windows）
├── record_ego_app.py               设备、网页服务和录制任务编排
├── record_ego_page.html            实时监看与控制页面
├── tracker_sender_windows.py       Windows 端 tracker 发送程序（Wuji 方案）
├── visualize_session.py            录完后回看一条 session
├── record_ego/                     采集主程序：配置、录制控制、会话落盘、上传
├── devices/                        设备驱动：wuji_glove / manus_glove / vive_tracker / network_tracker / gopro_camera
├── network_bridge/protocol.py      TCP / MessagePack 分帧协议
├── ManusClient/                    MANUS UDP 发送端及运行库（Windows）
├── scripts/
│   ├── setup_wuji_gloves.sh        Wuji 手套网卡配置与检查
│   ├── start_tracker_sender.ps1    Windows 端 tracker 发送启动脚本
│   ├── wuji_launch.sh              Wuji 方案一键启动（按本机修改路径）
│   ├── rclone_linux.sh             上传数据（Linux）
│   └── rclone_win.bat              上传数据（Windows）
└── task_setting/example.json       任务列表示例
```

## 3.2 任务列表

`--task-setting` 指向一个 JSON 文件，内容是非空的任务名数组。页面上按顺序切换任务，任务名会写进 session 名和 `session.json`：

```json
["物料称量", "萃取", "开反应", "旋蒸", "烘箱取料", "过柱"]
```

## 3.3 Wuji 方案启动

**先启动 Linux 采集端：**

```bash
cd capture && source .venv/bin/activate
export EGO_TRACKER_TOKEN="<两端相同的密钥>"
python record_ego_wuji.py \
  --task-setting task_setting/example.json \
  --tracker-host 0.0.0.0 --tracker-port 8766 \
  --save-dir ./data
```

也可以改好 `scripts/wuji_launch.sh` 里的路径后一键启动。

**再启动 Windows tracker 发送端**（在 `capture/` 目录下）：

```powershell
.\scripts\start_tracker_sender.ps1 -LinuxHost <LINUX_IP> -Port 8766 -Token "<两端相同的密钥>"
# 或者
python tracker_sender_windows.py --host <LINUX_IP> --port 8766 --token "<两端相同的密钥>"
```

- `host / port / token` 两端必须一致。连接中断后 Windows 每 2 秒自动重连，Linux 端不会退出，视频和手套继续采集。
- Windows 只需要允许 Python 出站访问局域网，不用开入站端口。
- **Windows 终端也能控制录制**（复用同一条已认证的 TCP 连接）：按 `r` 开始（倒计时用 Linux 端 `--record-countdown` 的设置，可用 `-Countdown 3` / `--countdown 3` 覆盖），按 `e` 结束并保存，按 `Esc` 只关闭快捷键监听、位姿继续发送。不需要时加 `-NoKeyboard` / `--no-keyboard`。

## 3.4 MANUS 方案启动

```bash
python record_ego_manus.py \
  --task-setting task_setting/example.json \
  --manus-ip 0.0.0.0 --manus-port 9000
```

tracker 由本机 `pyopenxr` 驱动；没有 tracker 时可以加 `--no-tracker`，相机和手套照常采集。

## 3.5 控制页面

默认 `http://127.0.0.1:8770/`（要从别的电脑打开，加 `--host 0.0.0.0` 并放行端口）。页面上有相机监看、tracker 3D 位姿、手套状态、任务切换和录制按钮；键盘快捷键与终端相同：

| 键 | 作用 |
|---|---|
| `r` | 倒计时后开始新 session（倒计时默认 3 秒，可在页面临时修改） |
| `e` | 结束并保存当前 session |
| `d` | 倒计时中取消；录制中**终止并丢弃**，不落盘 |
| `q` | 退出；未保存的录制会丢弃 |

Wuji 页面默认显示相机监看画面，监看流限制为 640×360（`--preview-width / --preview-height`），不影响录制分辨率。

## 3.6 常用参数

| 参数 | 默认值 | 说明 |
|---|---|---|
| `--cam-device` | `0` | 采集卡索引或 `/dev/video*` 路径 |
| `--cam-width` / `--cam-height` / `--cam-fps` | `2560` / `1440` / `60` | 向采集卡请求的分辨率和帧率；实际值以采集卡输出为准，记录在 `session.json` |
| `--video-codec` / `--crf` / `--h264-preset` / `--h264-threads` | `h264` / `20` / `ultrafast` / `4` | 软件 H.264 实时编码 |
| `--no-camera` | | 不连接相机 |
| `--wuji-calibration` / `--no-wuji-calibration` | 默认 bundle | Wuji 手套标定 |
| `--tracker-host` / `--tracker-port` / `--tracker-token` | `0.0.0.0` / `8766` / 空 | Wuji 方案的 tracker 接收 |
| `--tracker-record-fps` | `120` | Linux 端 tracker 写入上限 |
| `--tracker-sample-fps` | `90` | MANUS 方案本机 tracker 采样率 |
| `--no-tracker` | | 不接 tracker |
| `--save-dir` | `./data` | session 保存根目录 |
| `--record-countdown` | `3` | 开始前倒计时（秒） |
| `--rclone-remote` / `--rclone-config` | 空 | 上传目标（可选） |
| `--storage-limit-gb` / `--storage-check-interval` | `50` / `30` 分钟 | 数据目录超过上限时自动上传并清理已校验的文件 |
| `--host` / `--web-port` | `127.0.0.1` / `8770` | 控制页面监听地址和端口 |

完整参数用 `python record_ego_wuji.py --help` / `python record_ego_manus.py --help` 查看。

## 3.7 上传

`scripts/rclone_linux.sh` / `scripts/rclone_win.bat` 是 `rclone copy --progress` 的薄封装：把脚本开头的 `LOCAL_PATH`（本地数据目录）和 `REMOTE_PATH`（远端路径）改成自己的，直接运行即可。需要过滤或上传后删除本地，改脚本里那一条 rclone 命令（`--include`、`move`）。

## 3.8 故障排查

| 现象 | 先查 |
|---|---|
| Linux 一直显示"等待 Tracker" | Windows 的 `--host` 填的是 Linux 局域网 IP（不是 127.0.0.1）；两端端口和 token 一致；`Test-NetConnection <LINUX_IP> -Port 8766`；Linux 防火墙；两台机器在同一网段；先启动的是 Linux 端 |
| Windows 没有 tracker 位姿 | 依赖装好；SteamVR 是当前 OpenXR Runtime；tracker 已配对并分配角色（不能是 Handheld object） |
| Linux 找不到 Wuji 手套 | 已安装 `wuji-sdk` 并关闭其它占用 SDK 的程序；`setup_wuji_gloves.sh --check`；左右手序列号与当前 SDK 用户的标定 |
