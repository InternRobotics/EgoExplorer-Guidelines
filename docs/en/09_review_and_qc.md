# 9. QC and three-pane review

[Back to contents](../index.md#english) · [中文](../zh/09_review_and_qc.md)

QC has two layers: **automatic checks** (left/right swaps, drift, glove pairing, residuals) run around projection and feed the gallery; the **three-pane review** is a human pass over each session, with video, SMPL-X mesh, and hands and touch at the same instant side by side.

Commands use the variables of 7.3 (`WORK`, `PYTHONPATH=src:scripts`) and run from the toolkit root; check results go to `$WORK/out/gallery/`, where the gallery reads them.

## 9.1 Automatic checks

Run these **before** projection (they only read the raw recordings). `check_tracker_lr.py` and `check_startup_stance.py` need a profile, so run them per batch:

```bash
G=$WORK/out/gallery; mkdir -p $G
B=<batch>; PROFILE=$WORK/profiles/<subject>.json
python scripts/check_tracker_lr.py      --sync-root $WORK/ego_data_sync --datasets $B --profile $PROFILE --output $G/tracker_lr.$B.json
python scripts/check_startup_stance.py  --sync-root $WORK/ego_data_sync --datasets $B --profile $PROFILE --output $G/startup_stance.$B.json
python scripts/check_glove_sync.py      --sync-root $WORK/ego_data_sync --datasets $B --output $G/glove_sync.$B.json
python scripts/check_hand_identity.py   --sync-root $WORK/ego_data_sync --datasets $B --output $G/hand_identity.$B.json
```

With several batches, concatenate the per-batch `tracker_lr.<batch>.json` (and `glove_sync.<batch>.json`) into one `tracker_lr.json` (`glove_sync.json`); they are plain JSON lists.

Run the drift check **after** projection (it uses the projection residuals):

```bash
python scripts/check_tracker_drift.py --sync-root $WORK/input/ego_data_sync --projected $WORK/out/projected --output $G/tracker_drift.json
```

| Check | Script | Test |
|---|---|---|
| Tracker L/R swap | `check_tracker_lr.py` | The head mount maps the camera tracker's world rotation into the head joint frame, whose +X column is the subject's left. Projecting `left − right` of each tracker pair onto that axis should stay positive for the whole recording. **Decided on the starting stance** (when the feet stay close together, "the frames where the limbs are widest apart" is unreliable; an older method missed ankle swaps because of this) |
| Tracker drift / detachment | `check_tracker_drift.py` | The body is rigid over minutes: head-to-wrist/ankle distances oscillate with motion but have no long-run trend. Reports `trend_m` (linear trend of that distance × duration) and `residual_growth_mm` (growth of that tracker's projection residual from the first to the last fifth) |
| Glove pairing and timing | `check_glove_sync.py` | MANUS glove roots carry a world orientation; their angular speed is cross-correlated with the wrist tracker's: lag at peak = timing offset, a stronger crossed pairing = swapped. Quaternions are resampled to 30 Hz before differencing (raw 220 Hz jitter otherwise causes many false "weak" verdicts) and angular speed is clipped at 20 rad/s against bad frames. Wuji roots have no orientation and are reported `unsupported` |
| Glove handedness | `check_hand_identity.py` | The glove skeleton has its own chirality, so each stream's hand can be determined absolutely, frame by frame |
| Starting stance | `check_startup_stance.py` | Whether the starting geometry is plausible (e.g. wrists metres apart = drift) |
| Fit residual | `tracker_diagnostics.npz` | Per-tracker median residual > 15 mm high, > 60 mm abnormal (overall p75 is only 4–12 mm) |

Notes:

- **A frozen tracker looks like a swap** (two constant poses differ by a constant vector); check displacement before deciding.
- The glove verdict is **relative to the tracker labels**. If the wrist trackers themselves are swapped, the glove verdict inverts: actual swap = `glove_swapped XOR wrist_tracker_swapped`.

## 9.2 Fixes

```bash
python scripts/fix_tracker_lr.py --report $G/tracker_lr.json --sync-root $WORK/ego_data_sync --output-root $WORK/ego_data_lr_fixed
python scripts/fix_glove_lr.py   --report $G/glove_sync.json --tracker-report $G/tracker_lr.json \
  --sync-root $WORK/ego_data_sync --output-root $WORK/ego_data_lr_fixed
```

- Both scripts accept `--dry-run` to list what would change.
- Corrected sessions go into the parallel tree `$WORK/ego_data_lr_fixed/<batch>/<sid>/`; **originals are never touched**. Other files are symlinked, so a corrected copy works directly as input.
- Tracker correction swaps the left/right tracker streams and stores `tracker_lr_correction` in the npz; glove correction only rewrites `session.json` (swapping the two gloves' `slot`) and records the evidence in `glove_lr_correction`.
- Then build the merged input view (7.5, step 2); projection and fusion read from it. If projection already ran, delete the corrected sessions' `references/`, `proposals/` and `projected/` results and rerun them.

## 9.3 Three-pane review pages

Each session gets a self-contained `<slug>.review.html` (`slug` = `session_<date>_<time>`, without the Chinese task name because Chrome refuses media whose file name contains it; add a suffix if two collide).

```bash
R=$WORK/out/review; mkdir -p $R
for npz in $WORK/out/unified_30hz/*.npz; do
  name=$(basename "$npz" .npz)
  slug=$(printf '%s' "$name" | sed -E 's/^(session_[0-9]+_[0-9]+).*/\1/')
  mp4=$(ls $WORK/ego_data_sync/*/"$name"/"$name".mp4 2>/dev/null | head -1)
  # 1) 360p / 15 fps copy of the video for in-page playback
  [ -n "$mp4" ] && [ ! -f $R/$slug.360.mp4 ] && ffmpeg -y -v error -i "$mp4" -vf "scale=-2:360" -r 15 \
      -c:v libx264 -preset veryfast -crf 32 -an -movflags +faststart $R/$slug.360.mp4
  # 2) payload (subsampled, delta-coded mesh) + 3) page
  python scripts/build_review_payload.py --unified "$npz" --output-dir $R --body-model-path models/smplx --max-frames 300
  [ "$name" != "$slug" ] && mv -f $R/$name.payload.json $R/$slug.payload.json && mv -f $R/$name.mesh.i8c.gz $R/$slug.mesh.i8c.gz
  v=(); [ -f $R/$slug.360.mp4 ] && v=(--video)
  python scripts/make_review_html.py $R/$slug.payload.json --slug $slug "${v[@]}"
done
```

The page has three panes, all from the same 30 Hz export, so the video frame, mesh pose and touch reading at a slider position are **the same instant by construction**, with no alignment guessed by the viewer:

1. **Egocentric video** (the 360p copy)
2. **Whole-body SMPL-X mesh** (10,475 vertices per frame; by default at most 300 uniformly subsampled frames, vertex stream delta-coded against the previous frame and quantised, embedded in the page) + the five trackers
3. **Hands and touch**: hand skeletons and the 17-region touch map, with contacting regions highlighted and named

Sessions whose projection failed can still get a page (from the `export_unified_nobody.py` export: video + touch, with the failure reason in the mesh pane) so the failure can be diagnosed.

> Note: the page embeds its data in the HTML (`const P = …`); editing the `.payload.json` beside it does nothing, rebuild the page instead. The review mesh has at most 300 frames, so long sessions play at 0.6–2 Hz; **the stutter is the subsampling, not the encoding**. For full-rate visualisation read the `unified_30hz` npz directly.

## 9.4 Gallery

```bash
python scripts/build_ego_gallery.py --batch-root $WORK/out --output-dir $WORK/out/gallery \
  --sync-root $WORK/input/ego_data_sync --body-model-path models/smplx --device cpu --title "EgoExplorer review"
```

`gallery/index.html` is a self-contained overview (opens with a double-click):

- One card per session: skeleton projections and the five measured tracker points in three frames of **fixed physical size** (1.5 m wide × 2.25 m tall, constant floor line), residual sparkline, root-path thumbnail and statistics. The frames deliberately do not auto-scale, which would hide problems such as "the body sank into the floor" or "the scale blew up".
- Sort by residual / rotation / displacement and filter on every QC flag; failed sessions are listed with their cause.
- Every card links to its three-pane page (in `../review/` by default).

## 9.5 What to look for in the human pass

| Look at | Normal | If not, check first |
|---|---|---|
| Body vs video sync | Turns, reaches and squats happen together with the video | Timestamp conversion, `video_time_offset_s` |
| Left/right | Picking something up with the right hand in the video uses the right hand on the mesh | Tracker / glove swap (9.1) |
| Feet | Support foot neither sinks, floats nor slides | Floor estimate, L3 weight, torso lean limit |
| Bends, squats, sitting | The torso really leans instead of the whole body sinking | Torso lean limit pinned at 15° |
| Fingers | Flex toward the palm, never backwards | Glove side, left-hand mirror signs |
| Touch | Contact regions match where and when the hand touches objects in the video | Glove side, touch health table |
| Residual curve | Smooth, millimetres to a centimetre or two | Strap slip, tracker detachment, wrong profile |

Record the verdict (pass / issue / re-record) per session in one table; it is what you select release data from later.
