# 7. 原始数据 → SMPL-X

[返回目录](../index.md) · [English](../en/07_smplx_reconstruction.md)

本章覆盖身体重建（Complete + Constrain）、手部融合，以及两种用法：**直接用我们的 checkpoint**（7.2–7.6）或**重新训练 proposal 网络**（7.7）。所有步骤都在一台机器上完成，有一块 NVIDIA GPU 最好；没有 GPU 也能跑（`--device cpu`），只是慢很多。

## 7.1 重建在做什么

<video src="../../assets/video/reconstruction.mp4" controls muted loop playsinline width="100%"></video>

*一条真实录制（做三明治，30 Hz）：五个带拖尾的坐标轴是 tracker 位姿，人体是解出的 SMPL-X，手上的颜色是手套测到的分区压力。*

每条 session 经过三个阶段，每一阶段的结果都落盘，便于单独检查：

| 阶段 | 做什么 | 产物（`$WORK/out/<阶段>/<sid>/`） |
|---|---|---|
| reference | 用 subject profile 把五个 tracker 位姿换成关节锚点（去掉安装外参） | `references/` |
| proposal（Complete） | AvatarPoser 五 tracker 版（官方三 tracker 模型扩到 90 通道）在 60 Hz 网格上给出整段全身初始轨迹，补出 tracker 看不到的 17 个关节 | `proposals/`（含 `contact_probability.npy`，是**左右脚触地概率**，与手套触觉无关） |
| projection（Constrain） | 可微 SMPL-X 投影，三层约束联合优化：L1 骨架（tracker 位置/旋转 + 关节限位）、L2 自身表面（防穿插）、L3 地面（所有顶点不穿地） | `projected/`：`smplx_profile_driven.npz/.json`、`tracker_diagnostics.npz`、`projection_diagnostics.json`、`pipeline_metadata.json` |

## 7.2 环境与模型

工具包代码：`src/five_tracker_smplx/`（库）+ `scripts/`（各阶段脚本）。

```bash
conda create -n egoexplorer python=3.10 -y && conda activate egoexplorer
pip install torch                                  # 按本机 CUDA 版本选择
pip install -e ".[fit,test,dataset,visualize]"     # 在工具包根目录执行
pip install --no-deps human_body_prior              # VPoser；加 --no-deps，避免它改动已装好的 torch
```

需要的模型文件（放在工具包根目录下）：

| 文件 | 来源 |
|---|---|
| `models/smplx/SMPLX_NEUTRAL.npz` | [SMPL-X 官网](https://smpl-x.is.tue.mpg.de/)（需注册，遵守其许可） |
| `models/vposer/V02_05/` | VPoser（[human_body_prior](https://github.com/nghorbani/human_body_prior)，模型在 SMPL-X 官网下载） |
| `ckpt/avatarposer_five_v3/model.pt` | 我们的 v3 proposal 网络，随工具包发布 |
| `outputs/calibration_motion_vposer_mano.npz` | 01–17 标定参考动作缓存，随工具包发布（也可用 `scripts/render_calibration_protocol.py --rebuild-motion-cache` 重建） |
| `outputs/external_models/avatarposer.pth` | [官方 AvatarPoser](https://github.com/eth-siplab/AvatarPoser)，只有重训练时需要 |

## 7.3 工作目录约定

下面所有命令都用这几个变量，并在工具包根目录下执行：

```bash
export WORK=/path/to/egoexplorer_work        # 工作目录
export CKPT=ckpt/avatarposer_five_v3/model.pt
export PYTHONPATH=src:scripts
```

```text
$WORK/
├── ego_data_sync/<batch>/<session>/         原始录制（第 6 章的五个文件）
├── ego_data_lr_fixed/<batch>/<session>/     左右修正副本（第 9 章自动生成）
├── input/ego_data_sync/<batch>/<session>/   合并输入视图：修正过的取修正副本，其余取原始
├── profiles/<batch>.json                    该 batch 的 subject profile（第 5 章；每位受试者、每次穿戴一份）
└── out/                                     references / proposals / projected / unified_30hz / review / gallery
```

- **`<batch>` = 同一位受试者、同一次穿戴的一组录制**，整组用同一份 profile。不同受试者或重新穿戴后的录制放到不同的 batch。
- 目录名 `ego_data_sync`、`ego_data_lr_fixed` 不要改：检查和修正脚本靠这两个路径名识别 batch 名。
- 不要把任何结果写回原始录制目录。

## 7.4 v3 生产配置

| 项 | 值 |
|---|---|
| proposal 网络 | v3 `model.pt` |
| 投影脚本 | `scripts/project_sparse_tracker_baseline_l2.py` |
| L1 关节限位 | 膝 125° / 肘 140° / **躯干前倾 90°** / 膝反屈权重 100 / 骨盆滚转权重 100 |
| L2 表面 | `--surface-mode signed_top8 --mesh-collision-weight 3000 --mesh-collision-radius 0.018 --mesh-sample-count 256` |
| L3 地面 | 穿透惩罚 50 |
| 优化 | 60 次 Adam，**lr 0.004**，chunk 256 |

两条必须知道的理由：

- **lr 必须是 0.004**（不是脚本默认的 0.008）：0.008 在 20–35% 的真实 session 上会发散成 `FloatingPointError: non-finite SMPL-X parameters`。遇到发散，用一半学习率重跑那条 session。
- **躯干前倾上限 90°**（不是默认 15°）：15° 会让弯腰、弓步、坐下整个人沉到地板下。

## 7.5 运行

**一条命令跑完**（工具包的单机入口，依次执行下面各步以及第 8、9 章的导出、审阅页和 gallery，每步都跳过已完成的 session）：

```bash
WORK=/path/to/egoexplorer_work bash scripts/run_pipeline.sh all            # 所有 batch
WORK=/path/to/egoexplorer_work bash scripts/run_pipeline.sh project <batch> # 只跑某一步、某个 batch
WORK=/path/to/egoexplorer_work bash scripts/run_pipeline.sh status
```

投影发散的 session 会自动用 lr 0.002 重试一次。下面是各步分开执行的写法。


**第 1 步：左右检查与修正**（第 9.1–9.2 节）。先做这一步，投影直接用修正后的输入，省得重跑。

**第 2 步：建合并输入视图。** 要逐个文件建链接（`Path.rglob` 不会进入链接的目录，不能直接链接整个 session 目录）：

```bash
for d in "$WORK"/ego_data_sync/*/*/; do
  b=$(basename "$(dirname "$d")"); s=$(basename "$d")
  src="$WORK/ego_data_lr_fixed/$b/$s"; [ -d "$src" ] || src="${d%/}"
  mkdir -p "$WORK/input/ego_data_sync/$b/$s"
  for f in "$src"/*; do ln -sfn "$(readlink -f "$f")" "$WORK/input/ego_data_sync/$b/$s/$(basename "$f")"; done
done
```

**第 3 步：投影。** 每个 batch 用它自己的 profile 跑一次：

```bash
B=<batch>; PROFILE=$WORK/profiles/$B.json
python scripts/run_five_tracker_pipeline.py \
  --input-root $WORK/input/ego_data_sync/$B --subject-profile $PROFILE --checkpoint $CKPT \
  --output-root $WORK/out --body-model-path models/smplx --device cuda --workers 1 \
  --iterations 60 --learning-rate 0.004 --chunk-size 256 \
  --torso-lean-limit-deg 90 --knee-hyperextension-weight 100 --pelvis-roll-weight 100 \
  --projector-script scripts/project_sparse_tracker_baseline_l2.py --surface-mode signed_top8 \
  --mesh-collision-weight 3000 --mesh-collision-radius 0.018 --mesh-sample-count 256 \
  --skip-existing
```

- 加 `--session <session 名>`（可重复）只跑指定的 session。
- `--skip-existing` 支持断点续跑；单条失败不会中断整批，失败原因写在 `$WORK/out/pipeline_report.json`。
- `--workers` 是并行的 session 数。显存或内存不够时保持 1。进程被 OOM 杀掉不会留下 traceback：状态为 failed 却没有错误信息时，先查 `dmesg | tail`。
- 耗时参考：一条 3000 帧的 session，H200 上约 25 s；CPU 上要几分钟。

**`status=ok` 不等于质量好。** 筛数据要看 `tracker_diagnostics.npz` 的 `position_residual_m`：好的 session 是 3–12 mm，但也有 ok 的 session 残差几十到几百毫米。

**需要"tracker 约束 100%"时**（动态动作、标定录制、做教师标签）：用 hard 配置，即 300 次迭代、lr 0.01、`--position-weight 10000 --rotation-weight 60 --pose-prior-weight 0.5`，其余项不变。各动作窗口残差 1–4 mm，代价约为生产配置的 5 倍。汇报残差时按动作窗口看 p99/max，不要只看中位数（动态帧只占约 5%）。

## 7.6 手部融合

把手套骨架拟合到 MANO（SMPL-X 的手），再用标定好的腕部变换挂到解出来的手臂上，于是手指和触觉落在同一个身体、同一条时钟上。MANUS 25 点和 Wuji / 新 MANUS 21 点走同一条路径（按节点数选映射）。

```bash
python -m five_tracker_smplx.cli fuse-manus-batch \
  --input-root $WORK/input/ego_data_sync/$B --body-output-dir $WORK/out/projected \
  --subject-profile $PROFILE --body-model-path models/smplx --device cuda \
  --max-hand-frames 480 --iterations 60 --no-protocol --no-viewer
```

产物 `projected/<sid>/smplx_with_manus_hands.npz`：身体部分与融合前**完全一致**（`global_orient` 等不变），只填入 `left_hand_pose` / `right_hand_pose`（各 45 维，MANO 15 关节轴角）。参考精度：关节 RMS 约 8–12 mm。

- 手套左右以 `session.json` 的 `slot` 为准（修正副本里已改好）。
- Wuji 骨架没有手的全局朝向，手的朝向完全来自腕部 tracker；这是设计如此，不是 bug。
- 没有手套数据的 session，导出时 `hand_pose_source = none_neutral_zero`（中性零姿态占位，**不是测量值**）。

## 7.7 重新训练 proposal 网络

**什么时候需要：** 新动作类型的姿态在训练数据里不存在时，proposal 会给出"站立"姿态（例如抬腿、深蹲时膝髋几乎不弯），而 60 步投影追不回超过约 15° 的误差（Adam 每参数行程约 iters × lr = 0.24 rad）。**只是换了安装或换了人，不需要重训**，新标一份 profile 就行。

**生产配方（必须照做）：**

1. **起点是官方 AvatarPoser** `outputs/external_models/avatarposer.pth`，训练脚本把 54 个输入通道扩到 90（脚踝通道零初始化）。**不要**从之前的五 tracker `model.pt` 热启动。
2. **合成数据**（`--synthetic-root`，文件名 `sequence_*.npz`，键 `tracker_positions/rotations, global_orient, body_pose, contact`）：
   - VPoser 采样：`scripts/generate_sparse_tracker_synthetic.py --profile <profile> --sequences 24 --frames 240`
   - 引导姿态（B01–B10 全身动作库）用目标受试者 profile 转成合成序列（静态姿态 → 站–做–保持–站，两种速度；动态 → 两种时长重采样）
   - AMASS：`scripts/generate_amass_five_tracker_synthetic.py --amass <HDM05/CMU/BMLrub …> --profile <profile> --output <dir> --frames 240`（60 Hz，只取根 + 21 个身体关节）
3. **自然数据教师标签**（`--natural-root`，每条 `<sid>/smplx_profile_driven.{npz,json}`，npz 要带 `attachment_position/rotation`，json 要有 `source` 和 `ground_y_m`）：
   - 用 **hard 投影**（300 it，lr 0.01，pos 10000 / rot 60 / prior 0.5，躯干前倾 90°）打标签；
   - 再加**逐帧参考姿态先验**选对 IK 分支：`scripts/build_pose_reference.py` 生成参考（标定录制用 01–17 协议姿态库 + 段对齐；测试录制用 B01–B10 引导姿态），投影时 `--pose-reference <npz> --pose-reference-weight 30`（只作用于髋、脊柱、膝、踝、脚这些关节，手臂仍由 tracker 驱动）。不加这一步，深蹲、抬腿、弓步的标签会在多个 IK 解之间随机选。
4. **训练**：
   ```bash
   python scripts/train_avatarposer_five_tracker.py \
     --checkpoint outputs/external_models/avatarposer.pth \
     --synthetic-root <合成目录> --natural-root <教师标签目录> --holdout-token "" \
     --max-natural-windows $((2000 * <自然文件数>)) --hard-negative-ratio 0.3 \
     --epochs 3 --batch-size 32 --learning-rate 1e-4 --device cuda --output <输出目录>
   ```
   自然数据来自不同 profile 时不要传 `--mount-reference`（每个文件用自己的挂点）。
5. **评估**：在留出测试集上用**生产投影**（60 步）跑一遍，按动作窗口看各 tracker 残差和关节角；再拿一条以前的 session 做回归检查，确认老数据没变差。参考结果：重训后测试集平均残差从 35/25/20/30/53 mm（头/左手/右手/左脚/右脚）降到 8/2/2/3/5 mm。

v3 网络就是这样训练出来的：合成数据 = 24 条 VPoser 序列 + 20 条引导姿态序列 + AMASS（HDM05/CMU/BMLrub），自然数据 = 35 条带参考先验标签的录制 + 17 条按动作类别挑选的 hard 标签录制。

**新动作类型必须准备两样东西**：每个动作一个参考姿态，以及动作段/有效窗口的标注。没有它们，投影出的教师标签在多个 IK 分支之间是随机的。
