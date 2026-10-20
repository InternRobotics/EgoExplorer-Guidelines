# 12. Environment and pitfalls

[Back to contents](../index.md#english) · [中文](../zh/12_environment_and_pitfalls.md)

## 12.1 Running the whole pipeline on one machine

| Stage | Device | Notes |
|---|---|---|
| Calibration `calibrate-subject` | CPU | Once per profile |
| L/R checks and fixes, 30 Hz export, review pages, gallery | CPU | |
| Proposal + projection, hand fusion | GPU recommended | CPU works, much slower; the hard config costs about 5× production |
| Retraining | GPU | |

- Python 3.10 and one NVIDIA GPU are recommended. On Apple Silicon use `--device cpu` for the profile-driven temporal IK: optimiser divergence has been observed with some PyTorch/MPS combinations.
- Disk: tens of MB of output per session (excluding video); the 360p review copies come on top.
- Run long jobs inside `tmux` / `screen` or with `nohup`; every stage supports `--skip-existing` to resume.
- Environment notes:
  - Add packages to a working environment with `pip install --no-deps`; dependency resolution can otherwise replace torch.
  - Do not keep both `nvidia-*-cu12` and `nvidia-*-cu13` packages in one environment: they write the same files, and the wrong install order gives `undefined symbol: ncclCommResume`.
  - torch 2.x JIT-compiles triton kernels for some ops and needs the Python headers (`Python.h`); with a standalone Python build, add its `include/` to `CPATH`.
  - When several processes share a GPU, give each its own `TRITON_CACHE_DIR` / `TORCHINDUCTOR_CACHE_DIR`.

## 12.2 Cross-cutting pitfalls

1. **Sample rates never agree.** Resample by timestamp at every stage (`np.interp` / Slerp); never align by index or assume a fixed rate. `contact_probability[:count]` once assumed contact and trajectory had equal length: none of 844 sessions did, those below 60 Hz were silently misaligned and those above 60 Hz (a whole Wuji batch) crashed, 218 sessions. Reference implementation: `_load_contact_probability()`.
2. **`status=ok` is not quality.** Filter on `position_residual_m`.
3. **Relative vs absolute timestamps**: SMPL-X output starts at 0, every other stream is wall clock; mixing them is off by 1.7 billion seconds.
4. **Zero-norm quaternions marked valid**, and NaN in a few sessions: resample each tracker independently from `valid & finite` samples only.
5. **Non-monotonic timestamps**: a few files have two frames a few ms early, which crashes Slerp; the pipeline corrects them.
6. **Folder depth may vary**: always scan with `rglob`; fixed-depth globs silently miss sessions.
7. **Glove node count**: 21 nodes does not mean Wuji (new MANUS exports are 21-node too); decide the layout from `hand_skeleton.shape[1]`.
8. **lr 0.008 diverges**; use 0.004. A projection `FloatingPointError` can also come from the acos-at-±1 gradient in the joint-limit hinge (fixed), not from the learning rate.
9. **The default 15° torso lean limit sinks bends into the floor**; v3 uses 90°.
10. **OOM-killed subprocesses leave no traceback**: for `status: failed` without an error message, check `dmesg | tail` for `oom-kill` first.
11. **Never write results back into the capture folders**; write to `$WORK/out`.
12. **`_恢复` (recovery) recordings are systematically lower quality**, and some have wrist trackers that never recorded; review them separately when selecting data.
13. **Older MANUS recordings have trackers at ~15 Hz** (2.2.3; fixed in the recorder), so about half of their 30 Hz export is interpolated; check `upsample_ratio`.
14. **A wrong profile is expensive**: the same recording can go from about 60 mm to over 80 mm residual with another subject's profile. Keep one subject and one donning per batch.
