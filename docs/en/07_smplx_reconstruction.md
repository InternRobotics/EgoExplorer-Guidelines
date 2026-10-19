# 7. Raw data → SMPL-X

[Back to contents](../../README.md#english) · [中文](../zh/07_smplx_reconstruction.md)

This chapter covers body reconstruction (Complete + Constrain), hand fusion, and two ways to run it: **with our checkpoint** (7.2–7.5) or **after retraining the proposal network** (7.6).

## 7.1 What reconstruction does

Each session goes through three stages, each written to disk so it can be inspected on its own:

| Stage | What it does | Outputs (`$BATCH_ROOT/<stage>/<sid>/`) |
|---|---|---|
| reference | Uses the subject profile to turn the five tracker poses into joint anchors (mounting removed) | `references/` |
| proposal (Complete) | The five-tracker AvatarPoser (official three-tracker model extended to 90 channels) produces a first whole-body trajectory on a 60 Hz grid, supplying the 17 joints no tracker sees | `proposals/` (includes `contact_probability.npy`: **left/right foot-contact probability**, unrelated to glove touch) |
| projection (Constrain) | Differentiable SMPL-X projection optimising three layers jointly: L1 skeleton (tracker position/rotation + joint limits), L2 self-surface (no interpenetration), L3 floor (no posed vertex below the floor) | `projected/`: `smplx_profile_driven.npz/.json`, `tracker_diagnostics.npz`, `projection_diagnostics.json`, `pipeline_metadata.json` |

## 7.2 Environment (from the v3 archive)

The v3 archive `eailab-hdd2:/eailab/code/five_tracker_v3` contains everything v3 needs and rebuilds offline:

```bash
rclone copy eailab-hdd2:/eailab/code/five_tracker_v3 ./five_tracker_v3 --exclude "batch_v3/**"
cd five_tracker_v3
bash env/rebuild_env.sh /path/to/venv_root      # builds .venv_hcluster (CPU, torch 2.2.2) and .venv_hcluster_cuda (GPU, torch 2.14+cu130), all --no-index

export PROJECT=$PWD/code
export PYCPU=/path/to/venv_root/.venv_hcluster/bin/python
export PYCUDA=/path/to/venv_root/.venv_hcluster_cuda/bin/python
export CKPT=$PWD/ckpt/avatarposer_five_v3model_20260917_all/model.pt
export LAB_PROFILE=$PWD/ckpt/profiles/lab_subject_profile.json
export LFF_PROFILE=$PWD/ckpt/profiles/liangfanfan_subject_profile.json
ln -sfn $PWD/ckpt/models $PROJECT/models          # models/ is relative to PROJECT in the code
```

Checksums (`MANIFEST.txt`): `model.pt` sha256 `ebc04c60…`, `SMPLX_NEUTRAL.npz` `376021446…`.

Environment notes:

- Install anything into the cluster venvs with `pip install --no-deps`, otherwise torch gets replaced (this has happened).
- The `nvidia-*-cu12` and `-cu13` packages write the same files; reinstalling from the original freeze gives `undefined symbol: ncclCommResume`. The archive's `pins_cuda.txt` already drops cu12.
- GPU workers set private `TRITON_CACHE_DIR` / `TORCHINDUCTOR_CACHE_DIR` (the worker scripts use `/tmp/triton_cache_$$`).
- The T-cluster image has no Python 3.10: rebuild the venvs on python-build-standalone 3.10 and set `CPATH=<py310>/include/python3.10` (torch 2.14's bmm backward JITs a triton kernel that needs `Python.h`).

## 7.3 v3 production config (single source: `scripts/batch_v3_env.sh`)

| Item | Value |
|---|---|
| Proposal network | `avatarposer_five_v3model_20260917_all/model.pt` |
| Projector | `scripts/project_sparse_tracker_baseline_l2.py` |
| L1 joint limits | knee 125° / elbow 140° / **torso lean 90°** / knee hyperextension weight 100 / pelvis roll weight 100 |
| L2 surface | `--surface-mode signed_top8 --mesh-collision-weight 3000 --mesh-collision-radius 0.018 --mesh-sample-count 256` |
| L3 floor | penetration weight 50 |
| Optimisation | 60 Adam iterations, **lr 0.004**, chunk 256 |

Two values you must know the reason for:

- **lr must be 0.004** (not the script default 0.008): 0.008 diverges into `FloatingPointError: non-finite SMPL-X parameters` on 20–35% of real sessions. After one divergence the worker retries once at half the learning rate.
- **torso lean limit 90°** (not the default 15°): 15° makes bends, lunges and sit-downs sink through the floor.

## 7.4 Running a batch

```bash
cd $PROJECT
bash scripts/run_batch_v3.sh view      # build the filtered input view from SYNC_ROOT (per-file symlinks)
bash scripts/run_batch_v3.sh project   # GPU supervisor: reference → proposal → L1/L2/L3 projection
bash scripts/run_batch_v3.sh check     # tracker L/R / drift / glove pairing (chapter 9)
bash scripts/run_batch_v3.sh fix-lr    # write corrected copies, archive stale results
bash scripts/run_batch_v3.sh project   # rerun the corrected sessions
bash scripts/run_batch_v3.sh fuse      # hand fusion (7.5)
bash scripts/run_batch_v3.sh status    # per-stage counts at any time
```

Every path can be overridden through environment variables (`BATCH_ROOT`, `SYNC_ROOT`, `VIEW_ROOT`, `FIXED_ROOT`, `DATASETS`, `GPUS`, …). Workers claim sessions with an atomic `mkdir` lock, so more workers can join at any time; a failed session leaves a `projected/.failed_<sid>` marker and is not retried forever. Cluster submission details are in chapter 12.

**A single session** (for debugging):

```bash
PYTHONPATH=src:scripts $PYCUDA scripts/run_five_tracker_pipeline.py \
  --input-root <folder containing *.tracker.npz> --subject-profile $LAB_PROFILE --checkpoint $CKPT \
  --output-root outputs/debug_one --body-model-path models/smplx --device cuda --workers 1 \
  --iterations 60 --learning-rate 0.004 --chunk-size 256 \
  --torso-lean-limit-deg 90 --knee-hyperextension-weight 100 --pelvis-roll-weight 100 \
  --projector-script scripts/project_sparse_tracker_baseline_l2.py --surface-mode signed_top8 \
  --mesh-collision-weight 3000 --mesh-collision-radius 0.018 --mesh-sample-count 256 \
  --session <session name> --skip-existing
```

**`status=ok` does not mean good quality.** Filter on `position_residual_m` in `tracker_diagnostics.npz`: good sessions sit at 3–12 mm, but some sessions marked ok are at tens to hundreds of millimetres.

**When you need "100% tracker adherence"** (dynamic motion, calibration recordings, teacher labels), use the hard config: 300 iterations, lr 0.01, `--position-weight 10000 --rotation-weight 60 --pose-prior-weight 0.5`, everything else unchanged. Residuals drop to 1–4 mm per action window at about 5× the production cost. Report residuals per action window as p99/max, not medians (dynamic frames are only ~5% of a recording).

## 7.5 Hand fusion

The glove skeleton is fitted to MANO (SMPL-X's hands) and attached to the solved arm through the calibrated wrist transform, so fingers and touch land on the same body and the same clock. MANUS 25-node and Wuji / new MANUS 21-node data share one path (the mapping is picked by node count).

Batch: `bash scripts/run_batch_v3.sh fuse` (worker `fuse_hands_batch_v3.sh`, profile per session). One session:

```bash
PYTHONPATH=src:scripts $PYCUDA -m five_tracker_smplx.cli fuse-manus-batch \
  --input-root <session folder> --body-output-dir $BATCH_ROOT/projected \
  --subject-profile <profile for this session> --body-model-path models/smplx --device cuda \
  --max-hand-frames 480 --iterations 60 --no-protocol --no-viewer
```

Output `projected/<sid>/smplx_with_manus_hands.npz`: the body is **identical** to before fusion (`global_orient` etc. unchanged); only `left_hand_pose` / `right_hand_pose` (45 each, MANO 15-joint axis-angle) are filled in. Reference accuracy: about 8–12 mm joint RMS.

- Glove side comes from `slot` in `session.json` (already fixed in the corrected copies).
- Wuji skeletons have no global hand orientation, so hand orientation comes entirely from the wrist tracker. That is by design.
- Sessions without glove data export `hand_pose_source = none_neutral_zero` (a neutral zero-pose placeholder, **not a measurement**).

## 7.6 Retraining the proposal network

**When it is needed:** for a motion type absent from training, the proposal returns a standing pose (e.g. knees and hips barely bent during leg raises or squats), and 60 projection steps cannot recover more than ~15° (Adam's per-parameter travel is about iters × lr = 0.24 rad). **A new mounting or a new subject does not need retraining**; calibrate a new profile instead.

**Production recipe (follow it exactly):**

1. **Start from the official AvatarPoser** `outputs/external_models/avatarposer.pth` (sha256 `02f8888e…`). The trainer widens its 54 input channels to 90 (ankle channels zero-initialised). **Do not** warm-start from an earlier five-tracker `model.pt`.
2. **Synthetic data** (`--synthetic-root`, files named `sequence_*.npz` with keys `tracker_positions/rotations, global_orient, body_pose, contact`):
   - VPoser samples: `scripts/generate_sparse_tracker_synthetic.py --profile <profile> --sequences 24 --frames 240`
   - Guide poses (the B01–B10 whole-body library) converted with the target subject's profile (static poses → stand–pose–hold–stand at two speeds; dynamic → resampled at two durations)
   - AMASS: `scripts/generate_amass_five_tracker_synthetic.py --amass <HDM05/CMU/BMLrub …> --profile <profile> --output <dir> --frames 240` (60 Hz, root + 21 body joints only)
3. **Teacher labels for natural recordings** (`--natural-root`, one `<sid>/smplx_profile_driven.{npz,json}` each; the npz must carry `attachment_position/rotation`, the json `source` and `ground_y_m`):
   - Label with the **hard projection** (300 it, lr 0.01, pos 10000 / rot 60 / prior 0.5, torso lean 90°);
   - add a **per-frame pose-reference prior** to pick the right IK branch: `scripts/build_pose_reference.py` builds the reference (calibration recordings: the 01–17 protocol pose library + stage alignment; test recordings: the B01–B10 guides), and the projector takes `--pose-reference <npz> --pose-reference-weight 30` (applied to hips, spine, knees, ankles and feet only; arms stay tracker-driven). Without it, squat, leg-raise and lunge labels pick among IK branches at random.
4. **Train**:
   ```bash
   PYTHONPATH=src:scripts $PYCUDA scripts/train_avatarposer_five_tracker.py \
     --checkpoint outputs/external_models/avatarposer.pth \
     --synthetic-root <synthetic dir> --natural-root <teacher-label dir> --holdout-token "" \
     --max-natural-windows $((2000 * <number of natural files>)) --hard-negative-ratio 0.3 \
     --epochs 3 --batch-size 32 --learning-rate 1e-4 --device cuda --output <output dir>
   ```
   Do not pass `--mount-reference` when natural files come from different profiles (each file then uses its own mounts).
5. **Evaluate** on a held-out test set with the **production projection** (60 steps), looking at per-tracker residuals and joint angles per action window; then run one lab session as a regression check. Reference result: after retraining, mean test residuals went from 35/25/20/30/53 mm (head / left hand / right hand / left foot / right foot) to 8/2/2/3/5 mm.

The v3 network was trained this way: synthetic = 24 lab VPoser sequences + 20 guide-pose sequences + AMASS (HDM05/CMU/BMLrub, with collector B's profile); natural = 35 collector-B recordings (reference-prior labels) + 3 lab teachers + 14 lab sessions picked per category (hard labels). The full script is `scripts/worker_liangfanfan_v3model.sh`.

**A new motion type needs two things**: a reference pose for each action, and stage / active-window annotation. Without them the projector's teacher label is a coin flip between IK branches.
