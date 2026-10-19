# 7. Raw data → SMPL-X

[Back to contents](../index.md#english) · [中文](../zh/07_smplx_reconstruction.md)

This chapter covers body reconstruction (Complete + Constrain), hand fusion, and two ways to run it: **with our checkpoint** (7.2–7.6) or **after retraining the proposal network** (7.7). Everything runs on one machine. An NVIDIA GPU is best; without one it still runs (`--device cpu`), only much slower.

## 7.1 What reconstruction does

Each session goes through three stages, each written to disk so it can be inspected on its own:

| Stage | What it does | Outputs (`$WORK/out/<stage>/<sid>/`) |
|---|---|---|
| reference | Uses the subject profile to turn the five tracker poses into joint anchors (mounting removed) | `references/` |
| proposal (Complete) | The five-tracker AvatarPoser (official three-tracker model extended to 90 channels) produces a first whole-body trajectory on a 60 Hz grid, supplying the 17 joints no tracker sees | `proposals/` (includes `contact_probability.npy`: **left/right foot-contact probability**, unrelated to glove touch) |
| projection (Constrain) | Differentiable SMPL-X projection optimising three layers jointly: L1 skeleton (tracker position/rotation + joint limits), L2 self-surface (no interpenetration), L3 floor (no posed vertex below the floor) | `projected/`: `smplx_profile_driven.npz/.json`, `tracker_diagnostics.npz`, `projection_diagnostics.json`, `pipeline_metadata.json` |

## 7.2 Environment and models

Toolkit code: `src/five_tracker_smplx/` (library) + `scripts/` (stage scripts).

```bash
conda create -n egoexplorer python=3.10 -y && conda activate egoexplorer
pip install torch                                  # pick the build for your CUDA version
pip install -e ".[fit,test,dataset,visualize]"     # from the toolkit root
pip install --no-deps human_body_prior              # VPoser; --no-deps keeps it from touching the installed torch
```

Model files (under the toolkit root):

| File | Source |
|---|---|
| `models/smplx/SMPLX_NEUTRAL.npz` | [SMPL-X website](https://smpl-x.is.tue.mpg.de/) (registration required; respect its licence) |
| `models/vposer/V02_05/` | VPoser ([human_body_prior](https://github.com/nghorbani/human_body_prior); model downloaded from the SMPL-X website) |
| `ckpt/avatarposer_five_v3/model.pt` | Our v3 proposal network, released with the toolkit |
| `outputs/calibration_motion_vposer_mano.npz` | 01–17 calibration reference-motion cache, released with the toolkit (or rebuild with `scripts/render_calibration_protocol.py --rebuild-motion-cache`) |
| `outputs/external_models/avatarposer.pth` | [Official AvatarPoser](https://github.com/eth-siplab/AvatarPoser), only needed for retraining |

## 7.3 Work-folder convention

All commands below use these variables and run from the toolkit root:

```bash
export WORK=/path/to/egoexplorer_work        # work folder
export CKPT=ckpt/avatarposer_five_v3/model.pt
export PYTHONPATH=src:scripts
```

```text
$WORK/
├── ego_data_sync/<batch>/<session>/         raw recordings (the five files of chapter 6)
├── ego_data_lr_fixed/<batch>/<session>/     L/R-corrected copies (written automatically, chapter 9)
├── input/ego_data_sync/<batch>/<session>/   merged input view: corrected copy where one exists, otherwise raw
├── profiles/<subject>.json                  one subject profile per subject and donning (chapter 5)
└── out/                                     references / proposals / projected / unified_30hz / review / gallery
```

- **`<batch>` = recordings of one subject in one donning**, all processed with the same profile. A different subject, or the same subject after remounting, goes into a different batch.
- Keep the folder names `ego_data_sync` and `ego_data_lr_fixed`: the check and fix scripts read the batch name from them.
- Never write results back into the raw recording folders.

## 7.4 v3 production config

| Item | Value |
|---|---|
| Proposal network | v3 `model.pt` |
| Projector | `scripts/project_sparse_tracker_baseline_l2.py` |
| L1 joint limits | knee 125° / elbow 140° / **torso lean 90°** / knee hyperextension weight 100 / pelvis roll weight 100 |
| L2 surface | `--surface-mode signed_top8 --mesh-collision-weight 3000 --mesh-collision-radius 0.018 --mesh-sample-count 256` |
| L3 floor | penetration weight 50 |
| Optimisation | 60 Adam iterations, **lr 0.004**, chunk 256 |

Two values you must know the reason for:

- **lr must be 0.004** (not the script default 0.008): 0.008 diverges into `FloatingPointError: non-finite SMPL-X parameters` on 20–35% of real sessions. If a session diverges, rerun it at half the learning rate.
- **torso lean limit 90°** (not the default 15°): 15° makes bends, lunges and sit-downs sink through the floor.

## 7.5 Running it

**Step 1: left/right checks and fixes** (9.1–9.2). Do this first so projection reads the corrected input and nothing has to be rerun.

**Step 2: build the merged input view.** Link each file separately (`Path.rglob` does not descend into symlinked folders, so linking whole session folders does not work):

```bash
for d in "$WORK"/ego_data_sync/*/*/; do
  b=$(basename "$(dirname "$d")"); s=$(basename "$d")
  src="$WORK/ego_data_lr_fixed/$b/$s"; [ -d "$src" ] || src="${d%/}"
  mkdir -p "$WORK/input/ego_data_sync/$b/$s"
  for f in "$src"/*; do ln -sfn "$(readlink -f "$f")" "$WORK/input/ego_data_sync/$b/$s/$(basename "$f")"; done
done
```

**Step 3: projection**, once per batch with that batch's profile:

```bash
B=<batch>; PROFILE=$WORK/profiles/<subject>.json
python scripts/run_five_tracker_pipeline.py \
  --input-root $WORK/input/ego_data_sync/$B --subject-profile $PROFILE --checkpoint $CKPT \
  --output-root $WORK/out --body-model-path models/smplx --device cuda --workers 1 \
  --iterations 60 --learning-rate 0.004 --chunk-size 256 \
  --torso-lean-limit-deg 90 --knee-hyperextension-weight 100 --pelvis-roll-weight 100 \
  --projector-script scripts/project_sparse_tracker_baseline_l2.py --surface-mode signed_top8 \
  --mesh-collision-weight 3000 --mesh-collision-radius 0.018 --mesh-sample-count 256 \
  --skip-existing
```

- Add `--session <name>` (repeatable) to run specific sessions only.
- `--skip-existing` resumes an interrupted run; one failed session does not stop the batch, and failures are listed in `$WORK/out/pipeline_report.json`.
- `--workers` is the number of sessions in parallel; keep it at 1 if memory is tight. An OOM-killed process leaves no traceback: for "failed" without an error message, check `dmesg | tail` first.
- Timing: one 3000-frame session takes about 25 s on an H200, several minutes on a CPU.

**`status=ok` does not mean good quality.** Filter on `position_residual_m` in `tracker_diagnostics.npz`: good sessions sit at 3–12 mm, but some sessions marked ok are at tens to hundreds of millimetres.

**When you need "100% tracker adherence"** (dynamic motion, calibration recordings, teacher labels), use the hard config: 300 iterations, lr 0.01, `--position-weight 10000 --rotation-weight 60 --pose-prior-weight 0.5`, everything else unchanged. Residuals drop to 1–4 mm per action window at about 5× the production cost. Report residuals per action window as p99/max, not medians (dynamic frames are only ~5% of a recording).

## 7.6 Hand fusion

The glove skeleton is fitted to MANO (SMPL-X's hands) and attached to the solved arm through the calibrated wrist transform, so fingers and touch land on the same body and the same clock. MANUS 25-node and Wuji / new MANUS 21-node data share one path (the mapping is picked by node count).

```bash
python -m five_tracker_smplx.cli fuse-manus-batch \
  --input-root $WORK/input/ego_data_sync/$B --body-output-dir $WORK/out/projected \
  --subject-profile $PROFILE --body-model-path models/smplx --device cuda \
  --max-hand-frames 480 --iterations 60 --no-protocol --no-viewer
```

Output `projected/<sid>/smplx_with_manus_hands.npz`: the body is **identical** to before fusion (`global_orient` etc. unchanged); only `left_hand_pose` / `right_hand_pose` (45 each, MANO 15-joint axis-angle) are filled in. Reference accuracy: about 8–12 mm joint RMS.

- Glove side comes from `slot` in `session.json` (already fixed in the corrected copies).
- Wuji skeletons have no global hand orientation, so hand orientation comes entirely from the wrist tracker. That is by design.
- Sessions without glove data export `hand_pose_source = none_neutral_zero` (a neutral zero-pose placeholder, **not a measurement**).

## 7.7 Retraining the proposal network

**When it is needed:** for a motion type absent from training, the proposal returns a standing pose (e.g. knees and hips barely bent during leg raises or squats), and 60 projection steps cannot recover more than ~15° (Adam's per-parameter travel is about iters × lr = 0.24 rad). **A new mounting or a new subject does not need retraining**; calibrate a new profile instead.

**Production recipe (follow it exactly):**

1. **Start from the official AvatarPoser** `outputs/external_models/avatarposer.pth`. The trainer widens its 54 input channels to 90 (ankle channels zero-initialised). **Do not** warm-start from an earlier five-tracker `model.pt`.
2. **Synthetic data** (`--synthetic-root`, files named `sequence_*.npz` with keys `tracker_positions/rotations, global_orient, body_pose, contact`):
   - VPoser samples: `scripts/generate_sparse_tracker_synthetic.py --profile <profile> --sequences 24 --frames 240`
   - Guide poses (the B01–B10 whole-body library) converted with the target subject's profile (static poses → stand–pose–hold–stand at two speeds; dynamic → resampled at two durations)
   - AMASS: `scripts/generate_amass_five_tracker_synthetic.py --amass <HDM05/CMU/BMLrub …> --profile <profile> --output <dir> --frames 240` (60 Hz, root + 21 body joints only)
3. **Teacher labels for natural recordings** (`--natural-root`, one `<sid>/smplx_profile_driven.{npz,json}` each; the npz must carry `attachment_position/rotation`, the json `source` and `ground_y_m`):
   - Label with the **hard projection** (300 it, lr 0.01, pos 10000 / rot 60 / prior 0.5, torso lean 90°);
   - add a **per-frame pose-reference prior** to pick the right IK branch: `scripts/build_pose_reference.py` builds the reference (calibration recordings: the 01–17 protocol pose library + stage alignment; test recordings: the B01–B10 guides), and the projector takes `--pose-reference <npz> --pose-reference-weight 30` (applied to hips, spine, knees, ankles and feet only; arms stay tracker-driven). Without it, squat, leg-raise and lunge labels pick among IK branches at random.
4. **Train**:
   ```bash
   python scripts/train_avatarposer_five_tracker.py \
     --checkpoint outputs/external_models/avatarposer.pth \
     --synthetic-root <synthetic dir> --natural-root <teacher-label dir> --holdout-token "" \
     --max-natural-windows $((2000 * <number of natural files>)) --hard-negative-ratio 0.3 \
     --epochs 3 --batch-size 32 --learning-rate 1e-4 --device cuda --output <output dir>
   ```
   Do not pass `--mount-reference` when natural files come from different profiles (each file then uses its own mounts).
5. **Evaluate** on a held-out test set with the **production projection** (60 steps), looking at per-tracker residuals and joint angles per action window; then rerun an earlier session as a regression check. Reference result: after retraining, mean test residuals went from 35/25/20/30/53 mm (head / left hand / right hand / left foot / right foot) to 8/2/2/3/5 mm.

The v3 network was trained this way: synthetic = 24 VPoser sequences + 20 guide-pose sequences + AMASS (HDM05/CMU/BMLrub); natural = 35 recordings with reference-prior labels + 17 recordings picked per motion category with hard labels.

**A new motion type needs two things**: a reference pose for each action, and stage / active-window annotation. Without them the projector's teacher label is a coin flip between IK branches.
