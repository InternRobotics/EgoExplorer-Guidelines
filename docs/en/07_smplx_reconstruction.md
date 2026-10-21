# 7. From raw data to SMPL-X

[Back to contents](../index.md#english) · [中文](../zh/07_smplx_reconstruction.md)

This chapter turns raw recordings into SMPL-X with fingers: 7.1–7.6 use our trained model, and 7.7 explains how to retrain it. Everything runs on one machine. An NVIDIA GPU is best; a CPU also works (`--device cpu`), only much slower.

<video src="../../assets/video/reconstruction.mp4" controls muted loop playsinline width="100%"></video>

*One real recording (sandwich making, 30 Hz): the five trailing axes are the tracker poses, the body is the solved SMPL-X, and the colour on the hands is the pressure measured by the gloves in each region.*

## 7.1 Three stages

Each recording passes through three stages, and each stage's result is written to disk so it can be checked on its own.

| Stage | What it does | Output (`$WORK/out/<stage>/<recording>/`) |
|---|---|---|
| reference | Uses the calibration to turn the five tracker poses into joint positions (mounting removed) | `references/` |
| proposal (complete) | A five-tracker AvatarPoser gives a whole-body trajectory at 60 Hz, filling in the 17 joints no tracker sees | `proposals/`; its `contact_probability.npy` is the probability that each foot touches the ground, unrelated to glove touch |
| projection (constrain) | Differentiable optimisation of SMPL-X under three sets of constraints: L1 skeleton (tracker positions and rotations, joint limits), L2 body surface (no self-intersection), L3 floor (no vertex below the floor) | `projected/`: `smplx_profile_driven.npz` and `.json`, `tracker_diagnostics.npz`, `projection_diagnostics.json`, `pipeline_metadata.json` |

## 7.2 Environment and models

```bash
conda create -n egoexplorer python=3.10 -y && conda activate egoexplorer
pip install torch                                # pick the build for your CUDA version
pip install -e ".[all]"                          # from the toolkit root
```

Model files go under the toolkit root:

| File | Source |
|---|---|
| `models/smplx/SMPLX_NEUTRAL.npz` | [SMPL-X website](https://smpl-x.is.tue.mpg.de/); registration and licence required |
| `models/vposer/V02_05/` | VPoser, also from the SMPL-X website |
| `ckpt/avatarposer_five/model.pt` | The completion network, included in the repository |
| `assets/calibration_motion_vposer_mano.npz` | Calibration reference motion, included; can be regenerated with `scripts/render_calibration_protocol.py --rebuild-motion-cache` |
| `outputs/external_models/avatarposer.pth` | [Official AvatarPoser](https://github.com/eth-siplab/AvatarPoser) weights, only needed for retraining |

## 7.3 Work folder

All commands below run from the toolkit root and use these variables:

```bash
export WORK=/path/to/egoexplorer_work
export CKPT=ckpt/avatarposer_five/model.pt
export PYTHONPATH=src:scripts
```

```text
$WORK/
├── ego_data_sync/<batch>/<session>/         raw recordings (the five files of chapter 6)
├── ego_data_lr_fixed/<batch>/<session>/     L/R-corrected copies (written by 9.2)
├── input/ego_data_sync/<batch>/<session>/   merged input: the corrected copy where there is one, otherwise the original
├── profiles/<batch>.json                    calibration of this group (chapter 5)
└── out/                                     stage outputs, 30 Hz export, review pages, gallery
```

- **A batch is a group of recordings from one wearer in one donning**, all processed with one calibration. Recordings from another wearer or a new donning go into a new batch.
- Keep the folder names `ego_data_sync` and `ego_data_lr_fixed`; the check and fix scripts use them to identify batches.
- Never write results back into the raw recording folders.

## 7.4 Default configuration

The defaults of `scripts/run_five_tracker_pipeline.py` are the following and normally need no change:

| Item | Value |
|---|---|
| Completion network | `ckpt/avatarposer_five/model.pt` |
| Projector | `scripts/project_sparse_tracker_baseline_l2.py` |
| L1 joint limits | knee 125°, elbow 140°, torso lean 90°, knee hyperextension weight 100, pelvis roll weight 100 |
| L2 body surface | `signed_top8`, weight 3000, radius 1.8 cm, 256 samples |
| L3 floor | penetration weight 50 |
| Optimisation | 60 Adam steps, learning rate 0.004, 256 frames per chunk |

Leave these two unchanged:

- **Learning rate 0.004.** At 0.008, about 20–35% of real recordings diverge with `FloatingPointError: non-finite SMPL-X parameters`.
- **Torso lean limit 90°.** With a 15° limit, bending, lunging and sitting make the whole body sink below the floor.

## 7.5 Running it

### One command

```bash
WORK=/path/to/egoexplorer_work bash scripts/run_pipeline.sh all             # every batch
WORK=/path/to/egoexplorer_work bash scripts/run_pipeline.sh project <batch>  # one step, one batch
WORK=/path/to/egoexplorer_work bash scripts/run_pipeline.sh status          # progress per stage
```

It runs the side checks, the side fixes, the merged input, projection and hand fusion, then the export, review pages and gallery of chapters 8 and 9. Every step skips finished recordings, so an interrupted run simply resumes. A recording whose projection diverges is retried once at learning rate 0.002.

### Step by step

**Step 1: side checks and fixes** (9.1–9.2). Doing this first means projection reads the corrected input and nothing has to be redone.

**Step 2: merged input.** Link each file separately (`Path.rglob` does not enter linked folders, so linking whole recording folders does not work):

```bash
for d in "$WORK"/ego_data_sync/*/*/; do
  b=$(basename "$(dirname "$d")"); s=$(basename "$d")
  src="$WORK/ego_data_lr_fixed/$b/$s"; [ -d "$src" ] || src="${d%/}"
  mkdir -p "$WORK/input/ego_data_sync/$b/$s"
  for f in "$src"/*; do ln -sfn "$(readlink -f "$f")" "$WORK/input/ego_data_sync/$b/$s/$(basename "$f")"; done
done
```

**Step 3: projection**, once per batch with its own calibration:

```bash
B=<batch>
python scripts/run_five_tracker_pipeline.py \
  --input-root $WORK/input/ego_data_sync/$B --subject-profile $WORK/profiles/$B.json \
  --checkpoint $CKPT --output-root $WORK/out \
  --body-model-path models/smplx --device cuda --workers 1 --skip-existing
```

- `--session <recording>` (repeatable) processes only those recordings.
- A failed recording does not stop the batch; failures are listed in `$WORK/out/pipeline_report.json`.
- `--workers` is the number of recordings processed at once; keep it at 1 when memory is tight. A process killed for lack of memory leaves no error message: if a recording failed without a reason, check `dmesg | tail` first.
- Timing: a 3000-frame recording takes about 25 seconds on an H200 and a few minutes on a CPU.

**Look at residuals, not just status.** "ok" does not mean the result is good. Filter on `position_residual_m` in `tracker_diagnostics.npz`: good recordings sit at 3–12 mm, while some recordings marked ok are at tens to hundreds of millimetres.

**When the body must follow the trackers closely** (large movements, calibration recordings, training labels), use the strict configuration: 300 steps, learning rate 0.01, `--position-weight 10000 --rotation-weight 60 --pose-prior-weight 0.5`, everything else unchanged. Residuals per motion segment drop to 1–4 mm, at about 5× the run time. Dynamic motion is only about 5% of a recording, so report residuals per motion segment as p99 and maximum; the median hides the problem.

## 7.6 Hand fusion

The glove skeleton is first fitted to MANO (SMPL-X's hand model) and then attached to the solved arm through the calibrated wrist mounting, so fingers and touch end up on the same person and the same timeline as the body. MANUS and Wuji follow the same path; the mapping is chosen by node count.

```bash
python -m five_tracker_smplx.cli fuse-manus-batch \
  --input-root $WORK/input/ego_data_sync/$B/<session> --body-output-dir $WORK/out/projected \
  --subject-profile $WORK/profiles/$B.json --body-model-path models/smplx --device cuda \
  --max-hand-frames 480 --iterations 60 --no-protocol --no-viewer
```

The output is `projected/<recording>/smplx_with_manus_hands.npz`. The body is identical to before fusion; only `left_hand_pose` and `right_hand_pose` are added (45 values each, axis-angle of the 15 MANO joints). Joint fitting error is about 8–12 mm.

- The glove side comes from `slot` in `session.json`, already corrected in the corrected copies.
- Wuji skeletons carry no overall hand orientation, so the hand's orientation comes entirely from the wrist tracker.
- Recordings without glove data are exported with `hand_pose_source` set to `none_neutral_zero`: the hands are a neutral placeholder, not a measurement.

## 7.7 Retraining the completion network

**When it is needed.** For a kind of motion absent from training, the completion network returns something close to standing, for example knees and hips barely bent during leg raises or squats. The 60 projection steps cannot recover more than about 15° (each parameter moves at most about steps × learning rate = 0.24 rad), so the network has to be retrained. A new wearer or a new donning only needs a new calibration, not retraining.

**How to train:**

1. **Start from the official AvatarPoser.** Initialise from `outputs/external_models/avatarposer.pth`; the training script widens the input from 54 to 90 channels and initialises the new ankle channels to zero. Do not continue from an already trained five-tracker model.
2. **Synthetic data** (`--synthetic-root`, files named `sequence_*.npz`):
    - random VPoser poses: `scripts/generate_sparse_tracker_synthetic.py --profile <calibration> --sequences 24 --frames 240 --output <folder>`;
    - guide poses: the B01–B10 whole-body motions in `assets/guides/`, converted to synthetic sequences with the target wearer's calibration (static poses become stand–pose–hold–stand, dynamic motions are resampled at two durations; the conversion script is not yet in the toolkit);
    - AMASS: `scripts/generate_amass_five_tracker_synthetic.py --amass <HDM05/CMU/BMLrub …> --profile <calibration> --output <folder> --frames 240` (60 Hz, root and 21 body joints only).
3. **Labels for real recordings** (`--natural-root`, one `<recording>/smplx_profile_driven.{npz,json}` each):
    - project them with the strict configuration (7.5) and a torso lean limit of 90°;
    - add a per-frame reference-pose prior so the solve lands on the right pose. `scripts/build_pose_reference.py` builds the reference: calibration recordings use the 01–17 protocol poses (`assets/protocol_body_poses.json`) with the stage alignment, test recordings use the B01–B10 guide poses. Pass `--pose-reference <npz> --pose-reference-weight 30` to the projector; it applies only to hips, spine, knees, ankles and feet, and the arms still follow the trackers. Without it, labels for squats, leg raises and lunges jump between several possible poses.
4. **Train:**
    ```bash
    python scripts/train_avatarposer_five_tracker.py \
      --checkpoint outputs/external_models/avatarposer.pth \
      --synthetic-root <synthetic folder> --natural-root <label folder> --holdout-token "" \
      --max-natural-windows $((2000 * <number of real recordings>)) --hard-negative-ratio 0.3 \
      --epochs 3 --batch-size 32 --learning-rate 1e-4 --device cuda --output <output folder>
    ```
    When real recordings come from different calibrations, do not pass `--mount-reference`; each recording then uses its own mounting.
5. **Evaluate.** Run the default projection on held-out test recordings and compare per-tracker residuals and joint angles per motion segment, then check a few older recordings to make sure nothing got worse. For reference, retraining brought the mean test residuals from 35 / 25 / 20 / 30 / 53 mm (head, left hand, right hand, left foot, right foot) down to 8 / 2 / 2 / 3 / 5 mm.

Our model was trained this way, on 24 VPoser sequences, 20 guide-pose sequences and AMASS (HDM05, CMU, BMLrub) as synthetic data, plus 35 recordings with reference-pose priors and 17 recordings chosen per motion type as real data.

**A new kind of motion needs two things**: a reference pose for each action and the start and end of each motion segment. Without them, the projected labels pick among several possible poses at random.
