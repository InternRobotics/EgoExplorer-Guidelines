# 7. 原始数据 → SMPL-X

[返回目录](../../README.md) · [English](../en/07_smplx_reconstruction.md)

本章覆盖身体重建（Complete + Constrain）、手部融合，以及两种用法：**直接用我们的 checkpoint**（7.2–7.5）或**重新训练 proposal 网络**（7.6）。

## 7.1 重建在做什么

每条 session 经过三个阶段，每一阶段的结果都落盘，便于单独检查：

| 阶段 | 做什么 | 产物（`$BATCH_ROOT/<阶段>/<sid>/`） |
|---|---|---|
| reference | 用 subject profile 把五个 tracker 位姿换成关节锚点（去掉安装外参） | `references/` |
| proposal（Complete） | AvatarPoser 五 tracker 版（官方三 tracker 模型扩到 90 通道）在 60 Hz 网格上给出整段全身初始轨迹，补出 tracker 看不到的 17 个关节 | `proposals/`（含 `contact_probability.npy`，是**左右脚触地概率**，与手套触觉无关） |
| projection（Constrain） | 可微 SMPL-X 投影，三层约束联合优化：L1 骨架（tracker 位置/旋转 + 关节限位）、L2 自身表面（防穿插）、L3 地面（所有顶点不穿地） | `projected/`：`smplx_profile_driven.npz/.json`、`tracker_diagnostics.npz`、`projection_diagnostics.json`、`pipeline_metadata.json` |

## 7.2 准备环境（用 v3 归档）

v3 归档 `eailab-hdd2:/eailab/code/five_tracker_v3` 包含跑 v3 所需的全部东西，可离线重建：

```bash
rclone copy eailab-hdd2:/eailab/code/five_tracker_v3 ./five_tracker_v3 --exclude "batch_v3/**"
cd five_tracker_v3
bash env/rebuild_env.sh /path/to/venv_root      # 建 .venv_hcluster（CPU, torch 2.2.2）和 .venv_hcluster_cuda（GPU, torch 2.14+cu130），全程 --no-index

export PROJECT=$PWD/code
export PYCPU=/path/to/venv_root/.venv_hcluster/bin/python
export PYCUDA=/path/to/venv_root/.venv_hcluster_cuda/bin/python
export CKPT=$PWD/ckpt/avatarposer_five_v3model_20260917_all/model.pt
export LAB_PROFILE=$PWD/ckpt/profiles/lab_subject_profile.json
export LFF_PROFILE=$PWD/ckpt/profiles/liangfanfan_subject_profile.json
ln -sfn $PWD/ckpt/models $PROJECT/models          # 代码里 models/ 相对 PROJECT
```

校验和（`MANIFEST.txt`）：`model.pt` sha256 `ebc04c60…`，`SMPLX_NEUTRAL.npz` `376021446…`。

环境注意：

- 往集群 venv 里装任何包都用 `pip install --no-deps`，否则会把 torch 换掉（真实发生过）。
- `nvidia-*-cu12` 和 `-cu13` 两套包写同一批文件，照原 freeze 重装会导致 `undefined symbol: ncclCommResume`；归档里的 `pins_cuda.txt` 已经去掉了 cu12。
- GPU worker 设置私有的 `TRITON_CACHE_DIR` / `TORCHINDUCTOR_CACHE_DIR`（worker 脚本已用 `/tmp/triton_cache_$$`）。
- T 集群镜像没有 Python 3.10：用 python-build-standalone 3.10 重建 venv，并设 `CPATH=<py310>/include/python3.10`（torch 2.14 的 bmm backward 要 JIT triton kernel，需要 `Python.h`）。

## 7.3 v3 生产配置（唯一来源：`scripts/batch_v3_env.sh`）

| 项 | 值 |
|---|---|
| proposal 网络 | `avatarposer_five_v3model_20260917_all/model.pt` |
| 投影脚本 | `scripts/project_sparse_tracker_baseline_l2.py` |
| L1 关节限位 | 膝 125° / 肘 140° / **躯干前倾 90°** / 膝反屈权重 100 / 骨盆滚转权重 100 |
| L2 表面 | `--surface-mode signed_top8 --mesh-collision-weight 3000 --mesh-collision-radius 0.018 --mesh-sample-count 256` |
| L3 地面 | 穿透惩罚 50 |
| 优化 | 60 次 Adam，**lr 0.004**，chunk 256 |

两条必须知道的理由：

- **lr 必须是 0.004**（不是脚本默认的 0.008）：0.008 在 20–35% 的真实 session 上发散成 `FloatingPointError: non-finite SMPL-X parameters`。一次发散后 worker 会自动用一半学习率重试一次。
- **躯干前倾上限 90°**（不是默认 15°）：15° 会让弯腰、弓步、坐下整个人沉到地板下。

## 7.4 跑整批

```bash
cd $PROJECT
bash scripts/run_batch_v3.sh view      # 从 SYNC_ROOT 建过滤后的输入视图（逐文件符号链接）
bash scripts/run_batch_v3.sh project   # GPU supervisor：reference → proposal → L1/L2/L3 投影
bash scripts/run_batch_v3.sh check     # tracker 左右 / 漂移 / 手套配对（第 9 章）
bash scripts/run_batch_v3.sh fix-lr    # 写修正副本，归档旧结果
bash scripts/run_batch_v3.sh project   # 重跑被修正的 session
bash scripts/run_batch_v3.sh fuse      # 手部融合（7.5）
bash scripts/run_batch_v3.sh status    # 随时看各阶段计数
```

路径都可以用环境变量覆盖（`BATCH_ROOT`、`SYNC_ROOT`、`VIEW_ROOT`、`FIXED_ROOT`、`DATASETS`、`GPUS`…）。worker 用原子 `mkdir` 锁抢 session，可以随时增加 worker；失败的 session 会留下 `projected/.failed_<sid>` 标记，不会无限重试。集群提交细节见第 12 章。

**只跑一条 session**（调试用）：

```bash
PYTHONPATH=src:scripts $PYCUDA scripts/run_five_tracker_pipeline.py \
  --input-root <含 *.tracker.npz 的目录> --subject-profile $LAB_PROFILE --checkpoint $CKPT \
  --output-root outputs/debug_one --body-model-path models/smplx --device cuda --workers 1 \
  --iterations 60 --learning-rate 0.004 --chunk-size 256 \
  --torso-lean-limit-deg 90 --knee-hyperextension-weight 100 --pelvis-roll-weight 100 \
  --projector-script scripts/project_sparse_tracker_baseline_l2.py --surface-mode signed_top8 \
  --mesh-collision-weight 3000 --mesh-collision-radius 0.018 --mesh-sample-count 256 \
  --session <session 名> --skip-existing
```

**`status=ok` 不等于质量好。** 筛数据要看 `tracker_diagnostics.npz` 的 `position_residual_m`：好的 session 是 3–12 mm，但也有 ok 的 session 残差几十到几百毫米。

**需要"tracker 约束 100%"时**（动态动作、标定录制、要做教师标签）：用 hard 配置——300 次迭代、lr 0.01、`--position-weight 10000 --rotation-weight 60 --pose-prior-weight 0.5`，其余项不变。各动作窗口残差 1–4 mm，代价约为生产配置的 5 倍。汇报残差时按动作窗口看 p99/max，不要只看中位数（动态帧只占约 5%）。

## 7.5 手部融合

把手套骨架拟合到 MANO（SMPL-X 的手），再用标定好的腕部变换挂到解出来的手臂上，于是手指和触觉落在同一个身体、同一条时钟上。MANUS 25 点和 Wuji / 新 MANUS 21 点走同一条路径（按节点数选映射）。

批处理：`bash scripts/run_batch_v3.sh fuse`（worker `fuse_hands_batch_v3.sh`，按 session 选 profile）。单条：

```bash
PYTHONPATH=src:scripts $PYCUDA -m five_tracker_smplx.cli fuse-manus-batch \
  --input-root <session 目录> --body-output-dir $BATCH_ROOT/projected \
  --subject-profile <该 session 的 profile> --body-model-path models/smplx --device cuda \
  --max-hand-frames 480 --iterations 60 --no-protocol --no-viewer
```

产物 `projected/<sid>/smplx_with_manus_hands.npz`：身体部分与融合前**完全一致**（`global_orient` 等不变），只填入 `left_hand_pose` / `right_hand_pose`（各 45 维，MANO 15 关节轴角）。参考精度：关节 RMS 约 8–12 mm。

- 手套左右以 `session.json` 的 `slot` 为准（修正副本里已改好）。
- Wuji 骨架没有手的全局朝向，手的朝向完全来自腕部 tracker；这是设计如此，不是 bug。
- 没有手套数据的 session，导出时 `hand_pose_source = none_neutral_zero`（中性零姿态占位，**不是测量值**）。

## 7.6 重新训练 proposal 网络

**什么时候需要：** 新动作类型的姿态在训练数据里不存在时，proposal 会给出"站立"姿态（例如抬腿、深蹲时膝髋几乎不弯），而 60 步投影追不回超过 ~15° 的误差（Adam 每参数行程约 iters × lr = 0.24 rad）。**只是换了安装或换了人，不需要重训**，新标一份 profile 就行。

**生产配方（必须照做）：**

1. **起点是官方 AvatarPoser**：`outputs/external_models/avatarposer.pth`（sha256 `02f8888e…`），训练脚本把 54 个输入通道扩到 90（脚踝通道零初始化）。**不要**从之前的五 tracker `model.pt` 热启动。
2. **合成数据**（`--synthetic-root`，文件名 `sequence_*.npz`，键 `tracker_positions/rotations, global_orient, body_pose, contact`）：
   - VPoser 采样：`scripts/generate_sparse_tracker_synthetic.py --profile <profile> --sequences 24 --frames 240`
   - 引导姿态（B01–B10 全身动作库）用目标受试者 profile 转成合成序列（静态姿态 → 站–做–保持–站，两种速度；动态 → 两种时长重采样）
   - AMASS：`scripts/generate_amass_five_tracker_synthetic.py --amass <HDM05/CMU/BMLrub …> --profile <profile> --output <dir> --frames 240`（60 Hz，只取根 + 21 个身体关节）
3. **自然数据教师标签**（`--natural-root`，每条 `<sid>/smplx_profile_driven.{npz,json}`，npz 要带 `attachment_position/rotation`，json 要有 `source` 和 `ground_y_m`）：
   - 用 **hard 投影**（300 it，lr 0.01，pos 10000 / rot 60 / prior 0.5，躯干前倾 90°）打标签；
   - 再加**逐帧参考姿态先验**选对 IK 分支：`scripts/build_pose_reference.py` 生成参考（标定录制用 01–17 协议姿态库 + 段对齐；测试录制用 B01–B10 引导姿态），投影时 `--pose-reference <npz> --pose-reference-weight 30`（只作用于髋、脊柱、膝、踝、脚这些关节，手臂仍由 tracker 驱动）。不加这一步，深蹲、抬腿、弓步的标签会在多个 IK 解之间随机选。
4. **训练**：
   ```bash
   PYTHONPATH=src:scripts $PYCUDA scripts/train_avatarposer_five_tracker.py \
     --checkpoint outputs/external_models/avatarposer.pth \
     --synthetic-root <合成目录> --natural-root <教师标签目录> --holdout-token "" \
     --max-natural-windows $((2000 * <自然文件数>)) --hard-negative-ratio 0.3 \
     --epochs 3 --batch-size 32 --learning-rate 1e-4 --device cuda --output <输出目录>
   ```
   自然数据来自不同 profile 时不要传 `--mount-reference`（每个文件用自己的挂点）。
5. **评估**：在留出测试集上用**生产投影**（60 步）跑一遍，按动作窗口看各 tracker 残差和关节角；再拿一条实验室 session 做回归检查，确认老数据没变差。参考结果：重训后测试集平均残差从 35/25/20/30/53 mm（头/左手/右手/左脚/右脚）降到 8/2/2/3/5 mm。

v3 网络就是这样训练出来的：合成数据 = 24 条实验室 VPoser 序列 + 20 条引导姿态 + AMASS（HDM05/CMU/BMLrub，用采集员 B 的 profile），自然数据 = 35 条采集员 B 录制（参考先验标签）+ 3 条实验室教师 + 14 条按类别挑的实验室 session（hard 标签）。完整脚本见 `scripts/worker_liangfanfan_v3model.sh`。

**新动作类型必须准备两样东西**：每个动作一个参考姿态，以及动作段/有效窗口的标注。没有它们，投影出的教师标签在多个 IK 分支之间是随机的。
