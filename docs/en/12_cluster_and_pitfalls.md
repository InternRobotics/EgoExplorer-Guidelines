# 12. Cluster operations and pitfalls

[Back to contents](../index.md#english) · [中文](../zh/12_cluster_and_pitfalls.md)

## 12.1 H cluster

- **Project**: `~/workspace/five_tracker_smplx`; two venvs: `.venv_hcluster` (CPU, torch 2.2.2+cpu) for views, checks, export, gallery and review pages; `.venv_hcluster_cuda` (torch 2.14+cu130) for proposal and projection.
- **Storage**: home is only 100 GB and fills up (symptoms: scp "dest open … Failure", "No space left on device"). Batch outputs go to `/data` (NFS, also mounted in rlaunch containers). When full, clear `~/.cache/pip` first.
- **Data sync**: bucket → `/data/ego_data_sync/<dataset>/` with `rclone copy --exclude "*.mp4"`. The rclone config only exists under one specific SSH user (the account without `+root`); with the wrong account the data looks missing.
- **GPU submission**: `rlaunch --charged-group=eailabagent_gpu --gpu=1 …` in the foreground (`--detach` does not run the command after `--`), backgrounded locally with `nohup setsid`.
  - Submit only to h200 nodes 0252–0255, one node per job (`--positive-tags=node/<name>` is an AND list), rotated by `NODES` / `node_tag_for()` in `batch_v3_env.sh`.
  - Container start occasionally fails with `nvidia-container-cli: ldcache error`; it is a node-level infrastructure issue and a launcher retry loop (up to 6 attempts) handles it.
  - Containers see `~/workspace` and `/data` but **not the login node's `/tmp`**; keep worker scripts on shared paths.
  - Stop with `brainctl stop process/<name> -n ailab-eailabagent`; pods cannot always be killed through brainctl, so **check for existing launchers before submitting**; duplicate workers clobber each other's outputs.
- **Starting remote background jobs from the Mac**: `ssh -f host 'nohup setsid cmd </dev/null >/dev/null 2>&1 & disown'`, otherwise the local ssh can hang; `setsid` also lets a later `kill -TERM -<pgid>` reach the whole process tree. `pkill -f <pattern>` inside `ssh '…'` kills the ssh shell itself; use a bracket pattern like `[x]yz`.
- **Timing** (H200, one 3000-frame session): projection (60 Adam iterations, chunk 256) about 25 s; the hard config about 5×.

## 12.2 T cluster (weekly batch)

- A dev box (1 × A800) plus clusterx submission to the T queue (14 nodes × 8 A800). `/data` is an AFS volume shared with jobs; the dev box has no general internet (pypi, internal mirrors and the bucket work).
- Submission: `clusterx run -J <name> --gpus-per-task N … --image <image> --mount PV_AFS:<volume id>:/data --no-env`, always with `--exclude` for a few known-bad nodes (list in `weekly_t.sh`). `clusterx log` needs `--worker`; job scripts should write their own logs to `/data`.
- Weekly batch: `/data/zhouyunsong/weekly_t.sh` runs every Saturday 08:00 Beijing time, triggered by a launchd job on the Mac (`~/ego_weekly/ego_weekly.py`): find bucket sessions not yet synced → sync (no mp4) → view → head-height profile labels → check + fix-lr → clusterx projection + hand fusion → export → review → gallery → copy into `ego_review/T1012`. Every stage skips finished sessions, so rerunning after an interruption resumes. Status file `weekly/status`, logs `weekly/<run>/run.log`.
- The bucket is read-only from T (403); outputs live only on T's `/data` and the Mac.
- launchd exits 78 (EX_CONFIG) when WorkingDirectory or log paths are on an external disk; keep them under `~`.

## 12.3 Cross-cutting pitfalls

1. **Sample rates never agree.** Resample by timestamp at every stage (`np.interp` / Slerp); never align by index or assume a fixed rate. `contact_probability[:count]` once assumed contact and trajectory had equal length: none of 844 sessions did, those below 60 Hz were silently misaligned and those above 60 Hz (the whole Wuji batch) crashed, 218 sessions. Reference implementation: `_load_contact_probability()`.
2. **`status=ok` is not quality.** Filter on `position_residual_m`.
3. **Relative vs absolute timestamps**: SMPL-X output starts at 0, every other stream is wall clock; mixing them is off by 1.7 billion seconds.
4. **Zero-norm quaternions marked valid**, and NaN in a few sessions: resample each tracker independently from `valid & finite` samples only.
5. **Non-monotonic timestamps**: a few files have two frames a few ms early, which crashes Slerp; the pipeline nudges them.
6. **Folder depth differs**: always scan with `rglob`; fixed-depth globs silently miss sessions.
7. **Glove node count**: 21 nodes does not mean Wuji (new MANUS exports are 21-node too); decide the layout from `hand_skeleton.shape[1]`.
8. **lr 0.008 diverges**; use 0.004. A projection `FloatingPointError` can also come from the acos-at-±1 gradient in the joint-limit hinge (fixed), not from the learning rate.
9. **The default 15° torso lean limit sinks bends into the floor**; v3 uses 90°.
10. **OOM-killed subprocesses leave no traceback**: for `status: failed` without an error message, check `dmesg | tail` for `oom-kill` first.
11. **`pip install` without `--no-deps` replaces torch**; the internal mirror returns 504 for a few packages permanently, and the archive contains wheels repacked from the venv by RECORD.
12. **Never write results back into the capture folders**; write to `outputs/` or `$BATCH_ROOT`.
13. **`_恢复` (recovery) recordings are systematically lower quality**, and some have wrist trackers that never recorded; review them separately when selecting data.
14. **MANUS-rig trackers run at ~15 Hz** (2.2), so about half of a 30 Hz export is interpolated; check `upsample_ratio`.
