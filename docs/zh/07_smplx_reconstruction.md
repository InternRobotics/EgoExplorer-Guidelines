# 7. 从原始数据到 SMPL-X

[返回目录](../index.md) · [English](../en/07_smplx_reconstruction.md)

本章说明如何从原始录制得到带手指的 SMPL-X：7.1–7.6 使用我们训练好的模型，7.7 介绍如何重新训练。所有步骤都可以在一台机器上完成。有 NVIDIA GPU 最好，没有也能用 CPU 跑（`--device cpu`），只是慢很多。

<video src="../../assets/video/reconstruction.mp4" controls muted loop playsinline width="100%"></video>

*一条真实录制（做三明治，30 Hz）：带拖尾的五个坐标轴是 tracker 位姿，人体是解出的 SMPL-X，手上的颜色是手套测到的各区压力。*

## 7.1 重建分三个阶段

每条录制依次经过三个阶段，每个阶段的结果都写到磁盘上，可以单独检查。

| 阶段 | 做什么 | 输出（`$WORK/out/<阶段>/<录制名>/`） |
|---|---|---|
| reference | 用标定结果把五个 tracker 的位姿换算成关节位置（去掉安装参数） | `references/` |
| proposal（补全） | 五 tracker 版 AvatarPoser 在 60 Hz 下给出整段全身轨迹，补出 tracker 看不到的 17 个关节 | `proposals/`，其中 `contact_probability.npy` 是左右脚着地概率，与手套触觉无关 |
| projection（约束） | 对 SMPL-X 做可微优化，同时满足三层约束：L1 骨架（tracker 位置和旋转、关节限位），L2 身体表面（不自相穿插），L3 地面（所有顶点不穿地） | `projected/`：`smplx_profile_driven.npz` 和 `.json`、`tracker_diagnostics.npz`、`projection_diagnostics.json`、`pipeline_metadata.json` |

## 7.2 环境与模型

```bash
conda create -n egoexplorer python=3.10 -y && conda activate egoexplorer
pip install torch                                # 按本机 CUDA 版本选择
pip install -e ".[all]"                          # 在工具包根目录执行
```

模型文件放在工具包根目录下：

| 文件 | 来源 |
|---|---|
| `models/smplx/SMPLX_NEUTRAL.npz` | [SMPL-X 官网](https://smpl-x.is.tue.mpg.de/)，需注册并遵守其许可 |
| `models/vposer/V02_05/` | VPoser，同样从 SMPL-X 官网下载 |
| `ckpt/avatarposer_five/model.pt` | 补全阶段的网络，随仓库提供 |
| `assets/calibration_motion_vposer_mano.npz` | 标定参考动作，随仓库提供；也可以用 `scripts/render_calibration_protocol.py --rebuild-motion-cache` 重新生成 |
| `outputs/external_models/avatarposer.pth` | [官方 AvatarPoser](https://github.com/eth-siplab/AvatarPoser) 权重，只在重新训练时需要 |

## 7.3 工作目录

以下命令都在工具包根目录下执行，并用到这几个变量：

```bash
export WORK=/path/to/egoexplorer_work
export CKPT=ckpt/avatarposer_five/model.pt
export PYTHONPATH=src:scripts
```

```text
$WORK/
├── ego_data_sync/<batch>/<session>/         原始录制（第 6 章的五个文件）
├── ego_data_lr_fixed/<batch>/<session>/     左右修正副本（第 9.2 节自动生成）
├── input/ego_data_sync/<batch>/<session>/   合并后的输入：有修正副本的用副本，其余用原始文件
├── profiles/<batch>.json                    这一组的标定结果（第 5 章）
└── out/                                     各阶段输出、30 Hz 导出、校验页、总览页
```

- **一个 batch 就是同一个人、同一次佩戴的一组录制**，整组用同一份标定。换人或重新佩戴后的录制放进新的 batch。
- `ego_data_sync`、`ego_data_lr_fixed` 这两个目录名不要改，检查和修正脚本靠它们识别 batch。
- 任何结果都不要写回原始录制目录。

## 7.4 默认配置

`scripts/run_five_tracker_pipeline.py` 的默认参数就是下面这套配置，一般不需要改：

| 项 | 值 |
|---|---|
| 补全网络 | `ckpt/avatarposer_five/model.pt` |
| 投影脚本 | `scripts/project_sparse_tracker_baseline_l2.py` |
| L1 关节限位 | 膝 125°，肘 140°，躯干前倾 90°，膝反屈权重 100，骨盆侧倾权重 100 |
| L2 身体表面 | `signed_top8`，权重 3000，半径 1.8 cm，采样 256 点 |
| L3 地面 | 穿地惩罚权重 50 |
| 优化 | Adam 60 步，学习率 0.004，每段 256 帧 |

其中两项来自实际踩过的问题：

- **学习率 0.004。** 用 0.008 时，约 20–35% 的真实录制会发散，报 `FloatingPointError: non-finite SMPL-X parameters`。
- **躯干前倾上限 90°。** 上限设成 15° 时，弯腰、弓步、坐下这类动作会让整个人沉到地面以下。

## 7.5 运行

### 一条命令

```bash
WORK=/path/to/egoexplorer_work bash scripts/run_pipeline.sh all             # 处理所有 batch
WORK=/path/to/egoexplorer_work bash scripts/run_pipeline.sh project <batch>  # 只跑某一步、某个 batch
WORK=/path/to/egoexplorer_work bash scripts/run_pipeline.sh status          # 查看各阶段进度
```

它依次执行左右检查、左右修正、合并输入、投影、手部融合，以及第 8、9 章的导出、校验页和总览页。每一步都跳过已完成的录制，中断后重跑即可接着做。投影发散的录制会自动用学习率 0.002 重试一次。

### 分步执行

**第 1 步，左右检查与修正**（第 9.1–9.2 节）。先做这一步，投影时直接用修正后的输入，就不用返工。

**第 2 步，合并输入。** 每个文件单独建链接（`Path.rglob` 不会进入链接过去的目录，所以不能直接链接整个录制目录）：

```bash
for d in "$WORK"/ego_data_sync/*/*/; do
  b=$(basename "$(dirname "$d")"); s=$(basename "$d")
  src="$WORK/ego_data_lr_fixed/$b/$s"; [ -d "$src" ] || src="${d%/}"
  mkdir -p "$WORK/input/ego_data_sync/$b/$s"
  for f in "$src"/*; do ln -sfn "$(readlink -f "$f")" "$WORK/input/ego_data_sync/$b/$s/$(basename "$f")"; done
done
```

**第 3 步，投影。** 每个 batch 用自己的标定各跑一次：

```bash
B=<batch>
python scripts/run_five_tracker_pipeline.py \
  --input-root $WORK/input/ego_data_sync/$B --subject-profile $WORK/profiles/$B.json \
  --checkpoint $CKPT --output-root $WORK/out \
  --body-model-path models/smplx --device cuda --workers 1 --skip-existing
```

- `--session <录制名>`（可重复）只处理指定的录制。
- 单条失败不会中断整批，失败原因记录在 `$WORK/out/pipeline_report.json`。
- `--workers` 是同时处理的录制数，显存或内存紧张时保持 1。进程因内存不足被系统杀掉时不会留下报错信息：状态为失败却没有原因时，先用 `dmesg | tail` 查看。
- 耗时参考：3000 帧的录制在 H200 上约 25 秒，CPU 上要几分钟。

**看残差，不要只看状态。** 状态为 ok 不代表结果好。用 `tracker_diagnostics.npz` 里的 `position_residual_m` 筛选：好的录制在 3–12 mm；有些状态为 ok 的录制残差在几十到几百毫米。

**需要严格贴合 tracker 时**（动作幅度大的录制、标定录制、给训练做标签），改用严格配置：300 步、学习率 0.01、`--position-weight 10000 --rotation-weight 60 --pose-prior-weight 0.5`，其余不变。各动作段的残差可降到 1–4 mm，耗时约为默认配置的 5 倍。动态动作只占录制的约 5%，统计残差时要按动作段看 p99 和最大值，只看中位数会掩盖问题。

## 7.6 手部融合

手套骨架先拟合到 MANO（SMPL-X 的手部模型），再通过标定好的手腕安装参数接到解出的手臂上，手指和触觉就和身体对齐到了同一个人、同一条时间轴上。MANUS 和 Wuji 走同一条流程，按骨架的节点数选择对应关系。

```bash
python -m five_tracker_smplx.cli fuse-manus-batch \
  --input-root $WORK/input/ego_data_sync/$B/<session> --body-output-dir $WORK/out/projected \
  --subject-profile $WORK/profiles/$B.json --body-model-path models/smplx --device cuda \
  --max-hand-frames 480 --iterations 60 --no-protocol --no-viewer
```

输出 `projected/<录制名>/smplx_with_manus_hands.npz`。身体部分与融合前完全一致，只补上 `left_hand_pose` 和 `right_hand_pose`（各 45 维，MANO 15 个关节的轴角）。关节拟合误差约 8–12 mm。

- 手套左右以 `session.json` 里的 `slot` 为准，修正副本中已经改好。
- Wuji 骨架不含手的整体朝向，手的朝向完全由手腕 tracker 决定。
- 没有手套数据的录制，导出时 `hand_pose_source` 为 `none_neutral_zero`，手部是中性姿态的占位值，不是测量结果。

## 7.7 重新训练补全网络

**什么时候需要。** 如果某类动作在训练数据里没有出现过，补全网络会给出接近站立的姿态，例如抬腿、深蹲时膝和髋几乎不弯。投影的 60 步优化追不回超过约 15° 的偏差（每个参数最多移动约 步数 × 学习率 = 0.24 rad），这时需要重新训练。换人或重新佩戴只需要重新标定，不需要重新训练。

**训练方法：**

1. **从官方 AvatarPoser 出发。** 用 `outputs/external_models/avatarposer.pth` 初始化，训练脚本把输入从 54 通道扩到 90 通道，新增的脚踝通道初始化为 0。不要从已经训练过的五 tracker 模型继续训练。
2. **合成数据**（`--synthetic-root`，文件名 `sequence_*.npz`）：
    - VPoser 随机姿态：`scripts/generate_sparse_tracker_synthetic.py --profile <标定> --sequences 24 --frames 240 --output <目录>`；
    - 引导姿态：`assets/guides/` 里的 B01–B10 全身动作，用目标采集者的标定转换成合成序列（静态姿态按"站立—做动作—保持—站立"展开，动态动作按两种时长重采样；转换脚本尚未整理进工具包）；
    - AMASS：`scripts/generate_amass_five_tracker_synthetic.py --amass <HDM05/CMU/BMLrub …> --profile <标定> --output <目录> --frames 240`（60 Hz，只取根节点和 21 个身体关节）。
3. **真实录制的标签**（`--natural-root`，每条一个 `<录制名>/smplx_profile_driven.{npz,json}`）：
    - 用严格配置（7.5 节）投影出标签，躯干前倾上限 90°；
    - 加上逐帧的参考姿态先验，避免解到错误的姿态分支。`scripts/build_pose_reference.py` 生成参考姿态：标定录制用 01–17 的协议姿态（`assets/protocol_body_poses.json`）加上分段结果，测试录制用 B01–B10 引导姿态。投影时加 `--pose-reference <npz> --pose-reference-weight 30`，只作用于髋、脊柱、膝、踝和脚，手臂仍由 tracker 决定。不加这一步，深蹲、抬腿、弓步的标签会在几个可能的姿态之间随机跳。
4. **训练：**
    ```bash
    python scripts/train_avatarposer_five_tracker.py \
      --checkpoint outputs/external_models/avatarposer.pth \
      --synthetic-root <合成数据目录> --natural-root <标签目录> --holdout-token "" \
      --max-natural-windows $((2000 * <真实录制条数>)) --hard-negative-ratio 0.3 \
      --epochs 3 --batch-size 32 --learning-rate 1e-4 --device cuda --output <输出目录>
    ```
    真实录制来自不同标定时，不要传 `--mount-reference`，每条录制使用自己的安装参数。
5. **评估。** 在留出的测试录制上用默认配置跑投影，按动作段比较各 tracker 的残差和关节角，再挑几条旧录制确认没有变差。作为参考，重新训练后测试集的平均残差从 35 / 25 / 20 / 30 / 53 mm（头、左手、右手、左脚、右脚）降到了 8 / 2 / 2 / 3 / 5 mm。

我们的模型就是这样训练的。合成数据包括 24 条 VPoser 序列、20 条引导姿态序列和 AMASS（HDM05、CMU、BMLrub）；真实数据包括 35 条带参考姿态先验的录制和 17 条按动作类别挑选的录制。

**加入新的动作类型时，需要准备两样东西**：每个动作的参考姿态，以及动作段的起止标注。缺少它们，投影给出的标签会在几个可能的姿态之间随机选择。
