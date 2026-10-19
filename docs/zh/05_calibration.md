# 5. 标定与标定数据采集

[返回目录](../../README.md) · [English](../en/05_calibration.md)

## 5.1 为什么要单独标定

一个 tracker 的读数同时取决于两件未知的事：**它装在肢体的哪里、什么朝向**（安装外参），以及**肢体怎么动**（关节姿态）。两者一起拟合时，安装误差会被悄悄吸收成姿态误差。所以 C³ 的第一步是：**每次穿戴先用一段规定动作，把体型、尺度和五个安装外参解出来并固定**，之后所有任务录制都复用这份结果，不再改。

标定输出一个 `subject_profile.json`，内容是：

- SMPL-X `betas`（10 维）和米制 `scale`
- 五个 tracker 相对各自关节的固定安装变换（平移 + 旋转）
- 身体尺寸、个人步态/转身先验（来自 16–17 段）、逐 tracker 残差等质量信息

## 5.2 什么时候必须重新标定

- 换受试者
- 任何一个 tracker 被拆下重装、绑带明显滑动
- 更换手套或手背支架

只要安装不变，同一份 profile 可以跨天、跨任务复用。**安装没变时不需要重训练网络**：网络输入在进网络前已经去掉了安装外参（换 profile 验证过）。

## 5.3 标定动作（01–17，约 144 秒）

静态姿态每个至少稳定 3 秒；动态动作慢速、连续、幅度可控。数采员看 3D 参考动画照做：`outputs/calibration_protocol_reference.html`（可旋转、缩放、逐帧查看），或三视角 MP4。**以页面上标的"受试者左/右"为准，不要把屏幕左侧当成受试者左侧。**

| 段 | 时长 | 动作 | 用途 |
|---|---:|---|---|
| 01 | 7 s | 自然站立，脚尖向前，双臂自然下垂，目视前方 | 地面、身高、初始根节点 |
| 02 | 9 s | 身体不动，头缓慢左看、右看、低头、抬头，每端停 1 秒 | 头部安装旋转 |
| 03 | 7 s | A-pose：双臂约向下外展 45°，手心向下 | 肩宽、上臂 |
| 04 | 7 s | T-pose：双臂水平侧平举，肘伸直，手心向下 | 臂展、肩–腕长度 |
| 05 | 7 s | 双臂向前平举，肘伸直，手心向下 | 手背 tracker 位置与朝向 |
| 06 | 8 s | 侧平举，翻掌向上；按"右—左—右—左"依次屈肘两轮 | 上臂/前臂长度、手背外参 |
| 07 | 8 s | 保持前平举，手腕依次掌心向下、翻转、俯、仰 | 手背 tracker 旋转外参 |
| 08 | 8 s | 浅蹲两次，躯干尽量竖直，脚不动 | 髋–膝–踝长度、地面约束 |
| 09 | 7 s | 抬**受试者自己的**左腿后放下，两次，身体不侧倾 | 左踝外参 |
| 10 | 7 s | 抬受试者自己的右腿后放下，两次 | 右踝外参 |
| 11 | 7 s | 原地先抬脚跟、再抬脚尖 | 踝 tracker 旋转外参 |
| 12 | 7 s | 双手前举，五指张开、并拢 | 手掌尺度、手套静态偏置 |
| 13 | 7 s | 缓慢握拳、完全张开，两次 | 手指屈曲尺度与零位 |
| 14 | 9 s | 拇指依次碰食指、中指、无名指、小指，再张开 | 对掌与捏合 |
| 15 | 9 s | 食指、中指、无名指、小指依次完全屈伸 | 每根手指链的映射 |
| 16 | 16 s | 小步原地转身：先左转 180° 回正，再右转 180° 回正，头与躯干同向 | 世界 yaw、转身连续性、左右/正反验收 |
| 17 | 14 s | 面朝正前，向前迈两步，再向后退两步回到原位 | 步态、骨盆平移、地面接触验收 |

**采集要求：**

- 平整硬地面；用胶带标出每只脚的站位，脚尖朝同一方向。起始站位决定了世界坐标的参考左/前（第 11 章）。
- 开机静置，确认五个 tracker 在同一全局坐标系里、没有跳帧。
- 标定开始前录 3 秒空场或站立。
- 整段标定和后续任务中，tracker 不得拆装或移动。
- 16–17 是动态验收段：会进入整段重建、个人转身/步态先验和网页验收，但**不作为静态点进入尺度/体型方程**（转身和行走时骨盆一直在动，当静态点会污染体型和外参）。

## 5.4 运行标定（`calibrate-subject`）

在集群上用 CPU 环境即可：

```bash
cd ~/workspace/five_tracker_smplx
# --data：同一受试者、同一次穿戴的一条或多条 01–17 录制所在目录；身高、臂展单位为米
PYTHONPATH=src .venv_hcluster/bin/python -m five_tracker_smplx.cli calibrate-subject \
  --data <标定 session 根目录> \
  --height 1.60 --armspan 1.50 \
  --output-dir outputs/subject_calibration_<受试者>_<日期> \
  --body-model-path models/smplx \
  --vposer-path models/vposer/V02_05 \
  --motion-cache outputs/calibration_motion_vposer_mano.npz \
  --iterations 1100 --optimized-betas 5 --device cpu
```

- 动作对齐是自动的：01–11 用五个 tracker 的位置/旋转特征，12–15 再结合手套关节运动能量，16–17 用头和双踝的同步旋转和水平位移。每条记录都会输出 `protocol_alignment.json`（段边界、置信度）。新录制只有检测到真正的转身和行走才会追加 16–17。
- 自动对齐失败时可以人工标注段边界（`*_manual_alignment.json`），再参与标定。
- 可选 `--smplx-prior <npz>`：用视频恢复的 SMPL-X betas 作体型先验（例如 GVHMR 结果），只作先验，不做逐帧对齐。
- 拟合原则：五个 tracker 的平移/旋转外参整段只有一份；只释放前几个 beta，身高臂展作强约束（不要拿五个 tracker 自由优化全部 10 个 beta，那是不可观的）；骨盆–颈部相对重力只做软约束。

**产物**：`subject_profile.json`（格式 `five_tracker_smplx_subject_profile_v5_wxyz_source_columns`）+ `subject_calibration.html`（验收页）。

## 5.5 验收

在 `subject_calibration.html` 里检查：

- 01–17 标签顺序和动作边界正确；
- 头部转动时躯干不被带动；屈肘、手腕、浅蹲、左右抬腿、踝关节动作自然，左右无反转；
- 静态段双脚平放不滑；转体由交替小步完成；行走为前 2 步后 2 步，共 4 次落脚；
- 骨盆没有异常摇摆或相邻帧跳变；
- **静态段预测 tracker 与实测 tracker 的残差稳定，且不随动作系统性漂移。** 某个 tracker 残差长期高于约 3–4 cm 时，先查绑带是否滑动、左右角色是否正确，再查坐标系。

## 5.6 一条任务录制用哪份 profile

管线按 session 选 profile（`batch_v3_env.sh` 的 `profile_for_session()`），现有两份：

| profile | 文件（v3 归档 `ckpt/profiles/`） | 适用 |
|---|---|---|
| 采集员 A（lab） | `lab_subject_profile.json` | 早期 wuji 实验室任务 |
| 采集员 B | `liangfanfan_subject_profile.json` | manus_L1、manus_new，以及采集员 B 戴 Wuji 设备录的 session |

**不能只按数据集分。** 例如 wuji 0914–0918 的家务任务（三明治/叠衣服/纸箱/书架）其实是采集员 B 戴 Wuji 设备录的。判据是头部高度（头 tracker 到踝的高度差）：采集员 A 约 1.26–1.38 m，采集员 B 约 1.40–1.54 m。T 集群的周批处理用 `weekly_profile.py` 按 1.32 m ± 0.03 m 阈值加相邻 session 投票自动分类，结果写进 `profile_override.tsv`，`profile_for_session()` 优先读它。

用错 profile 的代价很明显（例：同一条三明治 session，lab profile 80 mm RMS，正确的 profile 62 mm）。**新受试者一定要新标一份 profile，并在采集记录里写清每条 session 是谁录的。**
