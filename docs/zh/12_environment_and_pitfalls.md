# 12. 运行环境与常见坑

[返回目录](../index.md) · [English](../en/12_environment_and_pitfalls.md)

## 12.1 一台机器跑完整条管线

| 阶段 | 设备 | 备注 |
|---|---|---|
| 标定 `calibrate-subject` | CPU | 每份 profile 一次 |
| 左右检查 / 修正、30 Hz 导出、审阅页、gallery | CPU | |
| proposal + 投影、手部融合 | GPU 推荐 | CPU 可用，慢很多；hard 配置约为生产配置的 5 倍 |
| 重训练 | GPU | |

- 推荐 Python 3.10 + 一块 NVIDIA GPU。Apple Silicon 上 profile 驱动的时序 IK 请用 `--device cpu`：在某些 PyTorch/MPS 组合上观察到过优化器发散。
- 磁盘：每条 session 的产出（不含视频）约几十 MB；审阅页的 360p 视频副本另算。
- 长任务放进 `tmux` / `screen` 或用 `nohup` 跑，所有阶段都支持 `--skip-existing` 断点续跑。
- 环境注意：
  - 往已装好的环境里加包用 `pip install --no-deps`，否则依赖解析可能把 torch 换掉。
  - 同一个环境里不要同时留着 `nvidia-*-cu12` 和 `nvidia-*-cu13` 两套包：它们写同一批文件，安装顺序不对就会出现 `undefined symbol: ncclCommResume`。
  - torch 2.x 在部分算子上会 JIT 编译 triton kernel，需要 Python 头文件（`Python.h`）；用独立打包的 Python 时把它的 `include/` 加进 `CPATH`。
  - 多个进程共用 GPU 时，给每个进程设置独立的 `TRITON_CACHE_DIR` / `TORCHINDUCTOR_CACHE_DIR`。

## 12.2 贯穿性的坑

1. **采样率永远不统一。** 任何阶段都按时间戳重采样（`np.interp` / Slerp），不按索引对齐、不假设固定帧率。曾经 `contact_probability[:count]` 假设 contact 与轨迹等长——844 条里没有一条等长，低于 60 Hz 的被静默错位截断，高于 60 Hz 的（整批 Wuji）直接崩了 218 条。参考实现：`_load_contact_probability()`。
2. **`status=ok` ≠ 质量好。** 按 `position_residual_m` 筛。
3. **相对 vs 绝对时间戳**：SMPL-X 输出从 0 开始，其它流是墙钟，直接混用会差 17 亿秒。
4. **零范数四元数仍标 valid**、个别 session 含 NaN：每个 tracker 独立重采样，只用 `valid & 有限值` 的样本。
5. **非单调时间戳**：个别文件有两帧早了几毫秒，Slerp 会崩；管线已做修正。
6. **目录深度可能不统一**：扫描一律 `rglob`，固定层数的 glob 会静默漏掉 session。
7. **手套节点数**：21 点不等于 Wuji（新 MANUS 也是 21 点），按 `hand_skeleton.shape[1]` 判布局。
8. **lr 0.008 会发散**，用 0.004；投影报 `FloatingPointError` 时也可能是关节限位里 acos 在 ±1 处的梯度问题（已修复），不是学习率问题。
9. **躯干前倾上限默认 15° 会让弯腰类动作沉到地下**，v3 用 90°。
10. **OOM 被杀的子进程没有 traceback**：`status: failed` 却没有错误信息时，先 `dmesg | tail` 查 `oom-kill`。
11. **不要把结果写回采集目录**，统一写到 `$WORK/out`。
12. **`_恢复` 录制质量系统性偏低**，其中一部分手腕 tracker 根本没记，挑数据时单独看。
13. **MANUS 方案 tracker 只有约 15 Hz**（第 2.2 节），30 Hz 导出里约一半是插值，看 `upsample_ratio`。
14. **profile 用错代价很大**：同一条录制，用错受试者的 profile 残差能从 60 mm 涨到 80 mm 以上。每个 batch 只放一位受试者、一次穿戴的录制。
