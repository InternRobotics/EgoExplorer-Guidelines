# 4. 如何采集数据

[返回目录](../index.md) · [English](../en/04_data_collection.md)

<video src="../assets/video/tasks.mp4" controls muted loop playsinline width="100%"></video>

*十类任务模板，同一套设备、同一个标定流程：上排六个实验室操作，下排四个家务和工业任务。*

## 4.1 一次采集日的流程

```text
穿戴（第 1 章） → 主机自检（第 2.1.4 / 2.2.5 节） → 录 01–17 标定（第 5 章） → 按任务列表逐条录制 → 回看抽查 → 上传（可选） → 填写采集记录
```

1. **穿戴并自检。**
2. **先录标定。** 每次穿戴都要录（同一次穿戴可以录 2–3 条备用）。标定 session 要能从名字区分出来（例如任务名用"标定"）。
3. **按任务列表录制**（启动方式见第 3 章）。页面上选好当前任务，按 `r` 开始、`e` 保存；做错了按 `d` 丢弃重来。
4. **回看抽查**：每天至少抽几条用 `visualize_session.py` 看一下（4.4 节）。
5. **上传（可选）**：本机存储不够时，配置云端对象存储和 rclone 后上传（第 3.7 节）。
6. **采集记录**：程序会在数据目录同级写 `recording_log_<日期>.csv`（每条 session 的起止时间、时长、各流帧数）。另外请人工记下每条 session 是哪位受试者、哪次穿戴录的，处理时要按这个分 batch。

## 4.2 命名与目录

```text
data/session_<YYYYMMDD>_<HHMMSS>_<任务名>/
├── session_….mp4
├── session_….video.npz
├── session_….npz
├── session_….tracker.npz
└── session_….session.json
```

- 例如 `session_20260901_154755_物料称量`。格式见第 6 章。
- "恢复"类录制（应对中断、失败后的纠正）在任务名后加 `_恢复`，例如 `session_20260901_164121_过柱_恢复`。这类录制是刻意采集的，用来覆盖人如何纠错。

## 4.3 录制规范

- **每段开头站定 2–3 秒**再开始动作，结尾也停 1–2 秒。投影和左右检查都用起始站姿做判据。
- **全程不碰 tracker 和绑带。** 一旦碰歪，在采集记录里注明，必要时重录标定。
- **手腕 tracker 必须在录。** 历史上部分 `_恢复` 录制只有视频有效、手腕 tracker 根本没动，这类 session 无法重建身体。
- **单段时长**：实验室任务通常 2–7 分钟，太长的录制可以拆成多段。
- **GoPro 屏显**：HDMI 输出会带上 GoPro 的屏显信息（电量、SD 卡状态等），出现在画面顶部，发布时会裁掉顶部 110 px。尽量不要让重要内容落在画面最上沿。

## 4.4 录完怎么检查

**看 `session.json`**：`video.queue_dropped_frames`、`read_failures`、`writer_failures` 应为 0；`tracker.dropped_frames` 应为 0；`overlap_s` 应接近录制时长。

**回看一条 session**：

```bash
python visualize_session.py data/session_YYYYMMDD_HHMMSS_TASK
# 可选：--no-open --port 8000 --max-fps 30 --build-only
```

它会自动识别手套类型（Wuji / MANUS，21 节点；旧版 25 节点 MANUS 也兼容），把视频、五个 tracker、双手骨架和触觉放到同一条时间轴上：

![一条 Wuji 录制的回看页面](../assets/img/capture_session_viewer.jpg)

*一条真实的 Wuji 物料称量录制：左边是 GoPro 画面（顶部能看到 GoPro 屏显）；右上是五个 tracker 在世界坐标中的位置；右下是左右手骨架和 24×31 触觉热力图。*

看什么：

- 右上角五个 tracker 都有坐标、都在动，转身时一起转；
- 两只手的骨架都在动，左右没有对调（视频里动的是哪只手，对应那一侧的骨架也在动）；
- Wuji：触觉热力图在抓握时亮起，大面积常亮或全黑说明手套或标定有问题；
- 时长和帧率与录制一致。
