# 8. 30 Hz 统一导出

[返回目录](../index.md) · [English](../en/08_unified_30hz.md)

## 8.1 为什么要统一导出

原始录制里没有两路信号的速率相同：tracker 15、65 或 90 Hz，手套 22、120 或 220 Hz，视频 30–60 Hz。投影得到的 SMPL-X 用的是 tracker 自己的时间戳，而且从 0 开始计时，其它信号用的却是主机的绝对时间。直接按下标把这些数组对在一起，结果是错的，而且不会报错。

`export_unified_30hz.py` 为每条录制输出一个 npz，**所有逐帧数组的第 i 个元素对应同一时刻**。时间栅格对齐到绝对时间的 1/30 秒整数倍，同一时刻在任何录制里都落在同一个位置。

## 8.2 运行

```bash
python scripts/export_unified_30hz.py --projected-root $WORK/out/projected \
  --output-root $WORK/out/unified_30hz --rate 30 --skip-existing
```

- 有手部融合结果时读 `smplx_with_manus_hands.npz`，没有时读 `smplx_profile_driven.npz`（手部为中性姿态）。
- `--session <录制名>` 只导出指定录制；`--keep-raw-tactile` 额外保存 744 个触点的原始值（每只手每条约 17 MB）。
- Wuji 触觉需要 `tactile/` 下的处理模块（`wuji_correct.py`、`wuji_layout.py`）和手套健康表（`health/health_db.npz`）。路径可以用环境变量 `EGOEXPLORER_TACTILE_DIR` 改。缺少这些文件时只导出原始值，并在报告里注明。

投影失败的录制可以单独导出视频和触觉，供第 9 章的校验页说明失败原因。**输出目录必须和上面分开**，因为这个脚本会删掉输出目录里已投影录制的导出：

```bash
python scripts/export_unified_nobody.py --sync-root $WORK/input/ego_data_sync \
  --batch-root $WORK/out --output-root $WORK/out/unified_30hz_nobody --skip-existing
```

## 8.3 重采样方式

| 信号 | 方法 | 原因 |
|---|---|---|
| 旋转 | Slerp | 在轴角或旋转矩阵上直接线性插值是错的 |
| 位置 | 线性插值 | |
| 有效标记 | 窗口内取"与" | |
| 触觉 | 窗口内取最大值；接触取"或" | 以 30 Hz 采样 120 Hz 的接触信号，会漏掉大部分短暂接触 |
| 视频 | 取最近一帧的帧号，并记录与栅格时刻的偏差 | 视频帧不能插值 |

每个 tracker 单独重采样，只用它自己的有效样本。超出某个 tracker 覆盖范围的栅格点记为 `tracker_valid=False`，不硬插值（长度为 0 的四元数会让 Slerp 直接报错）。每路信号还有一个 `*_interpolated` 标记：栅格点离最近的真实样本超过一个原生采样周期时为 True，用来区分测量值和跨越丢帧插出来的值。

往返误差（先重采样到 30 Hz，再插回原始时间戳）：MANUS 0.069° / 0.50 mm，Wuji 0.016° / 0.11 mm，触觉峰值全部保留，都远小于拟合误差本身。

## 8.4 输出内容：`unified_30hz/<录制名>.npz`

**时间**

| 键 | 形状 | 含义 |
|---|---|---|
| `timestamps` | (T,) float64 | 主机绝对时间（秒） |
| `rate_hz` | () | 30.0 |
| `upsample_ratio` | () | 30 ÷ tracker 原生速率。早期 MANUS 录制约 1.98，即一半是插值；Wuji 约 0.46 |

**身体与手部**（中性 SMPL-X，`num_betas=10`，`use_pca=False`）

| 键 | 形状 | 含义 |
|---|---|---|
| `betas`、`scale` | (10,)、() | 来自标定 |
| `global_orient`、`body_pose`、`transl` | (T,3)、(T,63)、(T,3) | 轴角（弧度）；`transl` 的含义见第 11.4 节 |
| `left_hand_pose`、`right_hand_pose` | (T,45) | MANO 15 个关节的轴角 |
| `hand_pose_source` | 字符串 | `glove_fused` 为手套测量；`none_neutral_zero` 为占位值 |
| `attachment_position`、`attachment_rotation` | (5,3)、(5,3,3) | 五个 tracker 相对关节的安装平移和旋转 |
| `body_interpolated` | (T,) bool | |
| `has_body` | () bool | 投影失败的导出为 False |
| `ground_y_m` | () | 地面在原始坐标中的 Y 值 |
| `coordinate_frame` | 字符串 | `raw_capture_y_up` |

**tracker**

| 键 | 形状 | 含义 |
|---|---|---|
| `tracker_roles` | (5,) | `camera, left_wrist, right_wrist, left_ankle, right_ankle`，与原始文件的顺序不同 |
| `tracker_position`、`tracker_rotation` | (T,5,3)、(T,5,3,3) | 位置（米）；旋转矩阵，各列是 tracker 自身坐标轴在世界中的方向 |
| `tracker_valid`、`tracker_interpolated`、`tracker_native_hz` | | |

**手套**

| 键 | 形状 | 含义 |
|---|---|---|
| `glove_layout` | 字符串 | `manus25` 或 `wuji21` |
| `glove_left_points`、`glove_right_points` | (T,N,3) | 手腕局部坐标系下的关节点，N 为 25 或 21 |
| `glove_*_native_hz`、`glove_*_interpolated` | | |

**触觉（仅 Wuji）**，`{S}` 为 `L` 或 `R`：

| 键 | 形状 | 含义 |
|---|---|---|
| `tactile_{S}_L4_contact` | (T,17) | 17 个区域的接触（0/1），窗口内取"或" |
| `tactile_{S}_L2_score` | (T,17) | 去基线后除以稳健标准差得到的分数 |
| `tactile_{S}_L1_residual` | (T,17) | 去掉滚动 10% 分位基线后的残差 |
| `tactile_{S}_coarse_contact`、`_coarse_score` | (T,6) | 拇指、食指、中指、无名指、小指、手掌 |
| `tactile_{S}_coverage`、`_sigma` | | 各区域可用触点的比例、噪声尺度 |
| `tactile_{S}_L0_raw` | (T,744) | 仅在 `--keep-raw-tactile` 或缺少健康表时导出 |
| `tactile_region_names`、`tactile_coarse_names`、`tactile_levels` | | 区域名称与各层定义 |

17 个区域：`thumb_distal, thumb_proximal, index_{distal,middle,proximal}, middle_{…}, ring_{…}, pinky_{…}, palm_radial, palm_central, palm_ulnar`。

触觉分层的定义和阈值写在 `wuji_correct.py` 开头：L0 原始值；L1 去掉滚动基线；L2 除以稳健标准差；L3 增益补偿（只用于消融实验，不导出）；L4 按区域 80% 分位并带迟滞判断，得到接触。手套老化后，靠的是"看变化量而不是绝对值"（L1、L2）恢复数据，增益补偿帮助不大。手套健康系数按录制估计，不能按天平均，因为很多线路时好时坏。

!!! note "说明"
    早期导出里有逐指平均值，键名也叫 `tactile_L0…L4`，含义与这里的 L0–L4 分层不同，不要混淆。

**视频**

| 键 | 形状 | 含义 |
|---|---|---|
| `video_frame_index` | (T,) int32 | 最近一帧的帧号 |
| `video_time_offset_s` | (T,) | 该帧与栅格时刻的实际偏差，实测最大约 22 ms |
| `video_native_hz` | () | |

同名的 `.json` 是摘要：帧数、各路原生速率、`body_interpolated_frac`、`tracker_valid_frac`、`hand_pose_source`、`video_max_offset_ms` 等。

## 8.5 注意事项

- `smplx_profile_driven.npz` 里的 `timestamps` 从 0 开始，要加上 `tracker_t_host[0]` 才是绝对时间。统一导出里已经换算好。
- 触觉的左右按设备名取：`glove_0` 为 L，`glove_1` 为 R，没有读修正副本里的 `slot`。目前所有 Wuji 设备都是 `glove_0` 戴在左手，已有的手套左右修正也都发生在没有触觉的 MANUS 数据上，所以没有影响。如果以后 Wuji 出现左右标反，这里要改成读 `slot`。
