# 12. Environment and common problems

[Back to contents](../index.md#english) · [中文](../zh/12_environment_and_pitfalls.md)

## 12.1 Running on one machine

| Stage | Device | Notes |
|---|---|---|
| Calibration `calibrate-subject` | CPU | Once per donning |
| Side checks and fixes, 30 Hz export, review pages, gallery, LeRobot conversion | CPU | |
| Completion and projection, hand fusion | GPU recommended | A CPU also works, much slower; the strict configuration takes about 5× as long |
| Retraining | GPU | |

- Python 3.10 and one NVIDIA GPU are recommended. On Apple Silicon use `--device cpu`: with some PyTorch versions the optimisation diverges on MPS.
- Disk: a few tens of MB of results per recording, excluding video; the 360p copies for review pages come on top.
- Run long jobs inside `tmux` or `screen`, or with `nohup`. Every stage skips finished recordings, so an interrupted run simply resumes.
- Environment:
    - add packages to a working environment with `pip install --no-deps`, otherwise dependency resolution may replace torch;
    - do not install both `nvidia-*-cu12` and `nvidia-*-cu13` in one environment; they write the same files, and the wrong install order gives `undefined symbol: ncclCommResume`;
    - some torch 2.x operators compile triton kernels on the fly and need the Python headers (`Python.h`); with a stand-alone Python build, add its `include/` folder to `CPATH`;
    - when several processes share a GPU, give each its own `TRITON_CACHE_DIR` and `TORCHINDUCTOR_CACHE_DIR`.

## 12.2 Where things go wrong

**The data**

1. **Streams never share a sample rate.** Resample by timestamp at every stage (linear interpolation or Slerp); never align by index and never assume a fixed frame rate. For example, foot-contact probability is at 60 Hz and the tracker trajectory at its native rate; truncating one to the other's length either silently misaligns them or fails outright. See `_load_contact_probability()` for the correct pattern.
2. **Relative and absolute time.** SMPL-X output starts at 0 while the other streams use the system clock; mixing them is off by more than a billion seconds.
3. **Invalid quaternions.** Some frames have zero-length quaternions while marked valid, and a few recordings contain NaN. Resample each tracker on its own from valid, finite samples only.
4. **Non-monotonic timestamps.** A few files have two frames a few milliseconds early, which makes Slerp fail; the pipeline handles this.
5. **Folder depth varies.** Always search recursively with `rglob`; fixed-depth globs silently miss recordings.
6. **Tell the glove type from `session.json`.** MANUS and Wuji skeletons both have 21 nodes, so check `glove_type` instead of the node count.
7. **Check recovery recordings separately**, making sure the wrist trackers have data.

**Processing parameters**

8. **Learning rate 0.008 diverges**; use 0.004.
9. **A 15° torso lean limit sinks bending motions below the floor**; the default configuration uses 90°.
10. **A wrong calibration is costly.** The same recording can go from about 60 mm to over 80 mm residual with someone else's calibration. Keep one wearer and one donning per batch.

**Running and troubleshooting**

11. **"ok" does not mean good.** Filter on `position_residual_m`.
12. **Processes killed for lack of memory leave no error message.** If a recording failed without a reason, check `dmesg | tail` for `oom-kill` first.
13. **Never write results back into capture folders**; write everything to `$WORK/out`.
