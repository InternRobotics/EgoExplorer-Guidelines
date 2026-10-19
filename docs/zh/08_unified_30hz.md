# 8. 30 Hz 统一导出

[返回目录](../index.md) · [English](../en/08_unified_30hz.md)

## 8.1 为什么要统一导出

原始录制里没有任何两个流共用采样率：tracker 15 或 65 Hz，手套 22/120/220 Hz，视频 30–60 Hz；SMPL-X 结果还写在 tracker 自己的时间戳上，而且是**相对第一帧**的，其它流都是**绝对主机墙钟**。直接按索引 zip 这些数组，结果是静默错误的。

`export_unified_30hz.py` 给每条 session 输出一个 npz：**所有逐帧数组的第 i 个元素代表同一时刻**。栅格锚定到绝对时间的 1/30 秒整数倍，所以同一时刻在任何 session 里都落在同一索引。

## 8.2 运行

```bash
bash scripts/run_batch_v3.sh export
# 等价于：
$PYCPU scripts/export_unified_30hz.py --projected-root $BATCH_ROOT/projected \
  --output-root $BATCH_ROOT/unified_30hz --rate 30 --skip-existing
```

- 可选 `--session <sid>`（可重复）、`--keep-raw-tactile`（额外保存 744 维原始触觉的窗口峰值，每手每条约 17 MB）。
- 优先读融合后的 `smplx_with_manus_hands.npz`，没有手部融合时退回 `smplx_profile_driven.npz`（手部为中性零姿态占位）。
- 投影失败的 session 用 `export_unified_nobody.py` 导出视频和触觉（无身体），并把失败原因写进导出，供审阅页说明。
- Wuji 触觉需要 `~/workspace/tactile_work/`（`wuji_correct.py`、`wuji_layout.py`、`health/health_db.npz`）。缺了它只能导出 L0 原始值，并在报告里注明"L0 only"。

## 8.3 重采样规则

| 信号 | 方法 | 原因 |
|---|---|---|
| 旋转 | Slerp | 在轴角或旋转矩阵上线性插值是错的 |
| 位置 | 线性插值 | |
| valid 掩码 | 窗口内逻辑与 | |
| 触觉 | **窗口内取峰值 / 接触取逻辑或** | 30 Hz 点采样 120 Hz 的接触信号会漏掉大部分短暂触碰 |
| 视频 | 最近帧号，并记录与栅格时刻的偏差 | 帧不能插值 |

每个 tracker **独立**重采样，只用自己的有效样本；栅格超出某个 tracker 覆盖范围处标 `tracker_valid=False`，不硬插（零范数四元数会让 Slerp 整条崩掉）。每路流还有 `*_interpolated` 掩码：离最近真实样本超过一个原生周期的栅格点为 True，用来区分测量值和跨丢帧空洞插出来的值。

往返验证（30 Hz 再插回原生时间戳）：MANUS 0.069° / 0.50 mm，Wuji 0.016° / 0.11 mm，触觉峰值 100% 保留，都远小于拟合残差本身。

## 8.4 输出键（`unified_30hz/<session>.npz`）

**时间轴**

| 键 | 形状 | 说明 |
|---|---|---|
| `timestamps` | (T,) float64 | 绝对主机墙钟（秒） |
| `rate_hz` | () | 30.0 |
| `upsample_ratio` | () | 30 ÷ tracker 原生率。MANUS 系 ≈ 1.98（一半是插值），Wuji ≈ 0.46（降采样） |

**身体 + 手部 SMPL-X**（中性 SMPL-X，`num_betas=10`，`use_pca=False`）

| 键 | 形状 | 说明 |
|---|---|---|
| `betas` / `scale` | (10,) / () | 来自 subject profile |
| `global_orient` / `body_pose` / `transl` | (T,3) / (T,63) / (T,3) | 轴角（弧度）；`transl` 的含义见第 11.4 节 |
| `left_hand_pose` / `right_hand_pose` | (T,45) | MANO 15 关节轴角 |
| `hand_pose_source` | 字符串 | `glove_fused`（测量值）或 `none_neutral_zero`（占位，不是测量值） |
| `attachment_position` / `attachment_rotation` | (5,3) / (5,3,3) | 五个 tracker 在关节系下的安装平移 / 旋转 |
| `body_interpolated` | (T,) bool | |
| `has_body` | () bool | 投影失败的导出为 False |
| `ground_y_m` | () | 地面在原始坐标中的 Y |
| `coordinate_frame` | 字符串 | `raw_capture_y_up` |

**Tracker**

| 键 | 形状 | 说明 |
|---|---|---|
| `tracker_roles` | (5,) | `camera, left_wrist, right_wrist, left_ankle, right_ankle`（注意顺序与原始 npz 不同） |
| `tracker_position` / `tracker_rotation` | (T,5,3) / (T,5,3,3) | 米 / 旋转矩阵（列 = tracker 自身轴在世界中的方向） |
| `tracker_valid` / `tracker_interpolated` / `tracker_native_hz` | | |

**手套**

| 键 | 形状 | 说明 |
|---|---|---|
| `glove_layout` | 字符串 | `manus25` 或 `wuji21` |
| `glove_left_points` / `glove_right_points` | (T,N,3) | **腕局部坐标系**下的关节点，N = 25 或 21 |
| `glove_*_native_hz` / `glove_*_interpolated` | | |

**触觉（仅 Wuji）**，`{S}` ∈ `L`、`R`：

| 键 | 形状 | 说明 |
|---|---|---|
| `tactile_{S}_L4_contact` | (T,17) | 17 区二值接触（窗口内逻辑或） |
| `tactile_{S}_L2_score` | (T,17) | 稳健 z 分数（除 MAD） |
| `tactile_{S}_L1_residual` | (T,17) | 去滚动 p10 基线后的残差 |
| `tactile_{S}_coarse_contact` / `_coarse_score` | (T,6) | 拇指/食指/中指/无名指/小指/掌心 |
| `tactile_{S}_coverage` / `_sigma` | | 每区可用 taxel 比例 / 噪声尺度 |
| `tactile_{S}_L0_raw` | (T,744) | 仅 `--keep-raw-tactile` 或缺 health 表时 |
| `tactile_region_names` / `tactile_coarse_names` / `tactile_levels` | | 名称与层级定义 |

17 区：`thumb_distal, thumb_proximal, index_{distal,middle,proximal}, middle_{…}, ring_{…}, pinky_{…}, palm_radial, palm_central, palm_ulnar`。

触觉分层（定义在 `wuji_correct.py`，阈值写在文件头）：L0 原始 → L1 去滚动 p10 基线 → L2 除以稳健 MAD 得 z 分数 → L3 增益补偿（只作消融，不用于发布）→ L4 区域 80 分位 + 双判据迟滞得到接触。**真正把衰减手套的数据救回来的是"用变化量代替绝对值"（L1/L2），不是增益补偿。** 健康系数按 session 估计，不能按天平均（很多线路时断时续）。旧导出用的是逐指均值 `tactile_L0…L4`，与这里的 L0–L4 层级不是一回事，注意区分。

**视频**

| 键 | 形状 | 说明 |
|---|---|---|
| `video_frame_index` | (T,) int32 | 最近视频帧号 |
| `video_time_offset_s` | (T,) | 该帧与栅格时刻的实际偏差（实测最大约 22 ms） |
| `video_native_hz` | () | |

旁边的 `<session>.json` 是摘要：帧数、各流原生率、`body_interpolated_frac`、`tracker_valid_frac`、`hand_pose_source`、`video_max_offset_ms` 等。

## 8.5 用的时候注意

- 不要拿 `smplx_profile_driven.npz` 的 `timestamps` 当绝对时间：它是相对第一帧的，`body_abs = body_rel + tracker_t_host[0]`。统一导出已经换算好了。
- 触觉左右按设备名 `glove_0 → L`、`glove_1 → R` 取，不读修正副本里的 `slot`。目前所有 Wuji 设备都是 glove_0 戴左手，所有手套左右修正也都发生在没有触觉的 MANUS 数据上，所以没有影响；但以后 Wuji 如果出现 slot 标反，这里需要改成读 `slot`。
