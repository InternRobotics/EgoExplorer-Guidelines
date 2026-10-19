# 12. 集群运行与常见坑

[返回目录](../../README.md) · [English](../en/12_cluster_and_pitfalls.md)

## 12.1 H 集群

- **项目**：`~/workspace/five_tracker_smplx`；两个 venv：`.venv_hcluster`（CPU，torch 2.2.2+cpu）用于视图、检查、导出、gallery、审阅页；`.venv_hcluster_cuda`（torch 2.14+cu130）用于 proposal 和投影。
- **存储**：home 只有 100 GB，容易写满（症状：scp "dest open … Failure"、"No space left on device"）。批处理产出放 `/data`（NFS，rlaunch 容器也挂载）。满了先清 `~/.cache/pip`。
- **数据同步**：桶 → `/data/ego_data_sync/<数据集>/`，用 `rclone copy --exclude "*.mp4"`。注意 rclone 配置只在特定 SSH 用户下（不带 `+root` 的那个账号）；用错账号会以为数据不存在。
- **GPU 提交**：`rlaunch --charged-group=eailabagent_gpu --gpu=1 …`，前台运行（`--detach` 不会执行 `--` 后面的命令），本地用 `nohup setsid` 放后台。
  - 只提交到 h200 节点 0252–0255，一个 job 钉一个节点（`--positive-tags=node/<name>` 是 AND 关系），由 `batch_v3_env.sh` 的 `NODES` / `node_tag_for()` 轮转。
  - 容器启动偶发 `nvidia-container-cli: ldcache error`，是节点基础设施问题，launcher 里重试最多 6 次即可。
  - 容器看得到 `~/workspace` 和 `/data`，**看不到登录节点的 `/tmp`**，worker 脚本要放在共享路径下。
  - 停止用 `brainctl stop process/<name> -n ailab-eailabagent`；pod 不一定能被 brainctl 杀掉，所以**提交前先检查有没有重复的 launcher**，重复的 worker 会互相覆盖输出。
- **从本地 Mac 起远程后台任务**：用 `ssh -f host 'nohup setsid cmd </dev/null >/dev/null 2>&1 & disown'`，否则本地 ssh 可能一直挂着；`setsid` 也让之后的 `kill -TERM -<pgid>` 能杀掉整棵进程树。在 `ssh '…'` 里用 `pkill -f <pattern>` 会把 ssh 自己的 shell 杀掉，用 `[x]yz` 这种括号写法。
- **耗时参考**（H200，单 session 3000 帧）：投影（60 次 Adam，chunk 256）约 25 s。hard 配置约为 5 倍。

## 12.2 T 集群（周批处理）

- 开发机（A800 ×1）+ clusterx 提交到 T 队列（14 节点 × 8 A800）。`/data` 是与 job 共享的 AFS 卷；开发机没有通用外网（pypi、内部镜像和桶可用）。
- 提交：`clusterx run -J <name> --gpus-per-task N … --image <镜像> --mount PV_AFS:<卷 id>:/data --no-env`，并始终 `--exclude` 掉几台已知有问题的节点（名单在 `weekly_t.sh`）。`clusterx log` 需要 `--worker`，job 脚本最好自己把日志写到 `/data`。
- 周批处理：`/data/zhouyunsong/weekly_t.sh` 每周六 08:00（北京时间）由 Mac 本地 launchd 触发（`~/ego_weekly/ego_weekly.py`）：找出桶里还没同步的新 session → 同步（不含 mp4）→ view → 按头部高度分 profile → check + fix-lr → clusterx 投影 + 手部融合 → export → review → gallery → 拷到 `ego_review/T1012`。每个阶段都跳过已完成的 session，中断后重跑即续跑。状态文件 `weekly/status`，日志 `weekly/<run>/run.log`。
- T 上的桶是只读的（403），产出只在 T 的 `/data` 和本地 Mac。
- launchd 的 WorkingDirectory / 日志路径放在外接盘上会以 78（EX_CONFIG）退出，放在 `~` 下。

## 12.3 贯穿性的坑

1. **采样率永远不统一。** 任何阶段都按时间戳重采样（`np.interp` / Slerp），不按索引对齐、不假设固定帧率。曾经 `contact_probability[:count]` 假设 contact 与轨迹等长——844 条里没有一条等长，低于 60 Hz 的被静默错位截断，高于 60 Hz 的（整个 Wuji 批次）直接崩了 218 条。参考实现：`_load_contact_probability()`。
2. **`status=ok` ≠ 质量好。** 按 `position_residual_m` 筛。
3. **相对 vs 绝对时间戳**：SMPL-X 输出从 0 开始，其它流是墙钟，直接混用会差 17 亿秒。
4. **零范数四元数仍标 valid**、个别 session 含 NaN：每个 tracker 独立重采样，只用 `valid & 有限值` 的样本。
5. **非单调时间戳**：个别文件有两帧早了几毫秒，Slerp 会崩；管线已做 nudge。
6. **目录深度不统一**：扫描一律 `rglob`，固定层数的 glob 会静默漏掉 session。
7. **手套节点数**：21 点不等于 Wuji（新 MANUS 也是 21 点），按 `hand_skeleton.shape[1]` 判布局。
8. **lr 0.008 会发散**，用 0.004；投影报 `FloatingPointError` 时还可能是关节限位里 acos 在 ±1 处的梯度问题（已修），不是学习率问题。
9. **躯干前倾上限默认 15° 会让弯腰类动作沉到地下**，v3 用 90°。
10. **OOM 被杀的子进程没有 traceback**：`status: failed` 却没有错误信息时，先 `dmesg | tail` 查 `oom-kill`。
11. **pip 装包不加 `--no-deps` 会替换 torch**；内部镜像对少数包长期 504，归档里已有从 venv 按 RECORD 重打包的 wheel。
12. **不要把结果写回采集目录**，统一写到 `outputs/` 或 `$BATCH_ROOT`。
13. **`_恢复` 录制质量系统性偏低**，其中一部分手腕 tracker 根本没记，挑数据时单独看。
14. **MANUS 方案 tracker 只有 ~15 Hz**（第 2.2 节），30 Hz 导出里约一半是插值；看 `upsample_ratio`。
