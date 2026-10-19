# 9. 质检与三联校验

[返回目录](../index.md) · [English](../en/09_review_and_qc.md)

质检分两层：**自动检查**（左右反、漂移、手套配对、残差）在投影前后跑，结果进 gallery；**三联校验**是人工逐条看：同一时刻的视频、SMPL-X mesh、手部与触觉并排显示。

## 9.1 自动检查（`run_batch_v3.sh check`）

| 检查 | 脚本 | 判据 |
|---|---|---|
| tracker 左右反 | `check_tracker_lr.py` | 头部安装外参把 camera tracker 的世界旋转换到头部关节系，其 +X 列是受试者的左方。把每对 tracker 的 `left − right` 投影到这个轴上，正常应全程为正。**用起始站姿判定**（双脚全程靠得近时，"肢体分得最开的帧"不可靠，旧方法曾因此漏掉脚踝反） |
| tracker 漂移 / 脱落 | `check_tracker_drift.py` | 身体在分钟尺度上是刚性的：头到腕/踝的距离随动作振荡但没有长期趋势。报告 `trend_m`（距离的线性趋势 × 时长）和 `residual_growth_mm`（该 tracker 投影残差从前 1/5 到后 1/5 的增长） |
| 手套配对与时序 | `check_glove_sync.py` | MANUS 手套根节点带世界朝向，与腕部 tracker 的角速度做互相关：峰值滞后 = 时序偏移，交叉配对相关更高 = 左右反。先把四元数重采样到 30 Hz 再差分（原始 220 Hz 抖动会造成大量误判），角速度截断到 20 rad/s 抗坏帧。Wuji 根节点无朝向，报 `unsupported` |
| 手套手性 | `check_hand_identity.py` | 手套骨架自带手性，逐帧判断每路数据来自哪只手，可以绝对确定 slot |
| 起始站姿 | `check_startup_stance.py` | 起始窗口几何是否合理（例如两手腕相距几米 = 漂移） |
| 拟合残差 | `tracker_diagnostics.npz` | 逐 tracker 残差中位数 > 15 mm 偏高、> 60 mm 异常（总体 p75 只有 4–12 mm） |

注意：

- **冻结的 tracker 会伪装成"左右反"**（两个常量位姿的差也是固定向量），判定前先看位移。
- 手套的判定是**相对 tracker 标签**的。如果腕部 tracker 本身左右反，手套结论要取反：实际交换 = `glove_swapped XOR wrist_tracker_swapped`。

## 9.2 修正（`run_batch_v3.sh fix-lr`）

- `fix_tracker_lr.py` / `fix_glove_lr.py` 把修正后的 session 写到平行目录 `FIXED_ROOT`（默认 `/data/ego_data_lr_fixed/<数据集>/<sid>/`），**原始数据不动**；其余文件用符号链接，修正副本可以直接作为输入根目录。
- tracker 修正交换左右 tracker 数据流，并在 npz 里写 `tracker_lr_correction`；手套修正只改写 `session.json`（交换两只手套的 `slot`），并记录证据 `glove_lr_correction`。
- 基于错误输入的旧结果被移到 `before_lr_fix/`，然后再跑一次 `project`，worker 会自动优先使用修正副本。

## 9.3 三联校验页（`review`）

```bash
bash scripts/run_batch_v3.sh videos    # 从桶里逐条拉 GoPro mp4，转 360p/15 fps 副本放到审阅页旁（不会同时占用几百 GB）
bash scripts/run_batch_v3.sh review    # 每条 session 生成一个自包含的三联页
bash scripts/run_batch_v3.sh gallery   # 总览页
```

每条 session 一个 `review/<slug>.review.html`（`slug` = `session_<日期>_<时间>`，不带中文任务名，因为 Chrome 拒绝加载带中文的媒体文件名；少数重名 session 用 `<slug>_<数据集>`，见 `review/slug_overrides.json`）。

页面三栏，全部来自同一个 30 Hz 导出，所以滑块位置对应的视频帧、mesh 姿态和触觉读数**在构造上就是同一时刻**，不需要 viewer 去猜对齐：

1. **第一视角视频**（360p 副本）
2. **SMPL-X 全身 mesh**（10,475 顶点/帧；默认均匀抽取不超过 300 帧，顶点流相对上一帧做差分，量化成 int16/int8 后内嵌在页面里）+ 五个 tracker
3. **双手与触觉**：手部骨架和 17 区触觉，接触区高亮并列出区名

投影失败的 session 也会生成页面（视频 + 触觉，mesh 栏写明失败原因），方便判断失败原因。

> 注意：审阅页把数据直接内嵌在 HTML 里（`const P = …`），改旁边的 `.payload.json` 不会生效，要重新生成页面。审阅页里的 mesh 最多 300 帧，长 session 只有 0.6–2 Hz，**看起来卡是数据抽稀导致的，不是编码问题**；要做全速率可视化请直接用 `unified_30hz` 的 npz。

## 9.4 Gallery

`gallery/index.html` 是单文件自包含的总览（双击即可打开）：

- 每条 session 一张卡片：3 个**固定物理尺寸**画框（宽 1.5 m × 高 2.25 m，地面线位置恒定）里的骨架投影 + 五个 tracker 实测点、残差曲线、根轨迹缩略图、统计量。画框故意不自动缩放，否则"人沉到地板下""尺度爆掉"这类问题会被藏起来。
- 可按残差 / 旋转 / 位移排序，按全部检查标注筛选；失败 session 也列出并注明原因。
- 每张卡片链接到该 session 的三联页。

## 9.5 人工复核看什么

| 看什么 | 正常 | 异常时先查 |
|---|---|---|
| 身体与视频是否同步 | 转身、伸手、下蹲与视频同时发生 | 时间戳换算、视频偏差 `video_time_offset_s` |
| 左右 | 视频里用右手拿东西，mesh 也是右手 | tracker / 手套左右反（9.1） |
| 脚 | 支撑脚不穿地、不漂浮、不滑 | 地面估计、L3 权重、躯干前倾上限 |
| 弯腰、深蹲、坐下 | 躯干真的前倾，而不是整个人下沉 | 躯干前倾上限是否卡在 15° |
| 手指 | 弯曲方向朝掌心，无反折 | 手套左右、左手镜像符号 |
| 触觉 | 接触区与视频里手接触物体的位置、时刻一致 | 手套左右、触觉健康表 |
| 残差曲线 | 平稳、量级在毫米到一两厘米 | 绑带滑动、tracker 脱落、profile 用错 |

复核结果（通过 / 有问题 / 需重录）建议记录到一张表里，与 session 名对应，后续挑选发布数据时用。
