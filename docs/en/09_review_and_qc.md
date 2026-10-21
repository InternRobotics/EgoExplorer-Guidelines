# 9. QC and three-pane review

[Back to contents](../index.md#english) · [中文](../zh/09_review_and_qc.md)

QC has two parts. **Automatic checks** run before and after projection; they look for swapped sides, drifting trackers and mismatched gloves, and their results feed the gallery. The **three-pane review** is a manual pass over each recording, with video, SMPL-X, and hands and touch at the same instant side by side.

The commands in this chapter run from the toolkit root with the variables of 7.3. Check results go to `$WORK/out/gallery/`, where the gallery reads them.

## 9.1 Automatic checks

These run **before** projection and only read the raw recordings. Two of them need the calibration, so run them per batch:

```bash
G=$WORK/out/gallery; mkdir -p $G
B=<batch>; P=$WORK/profiles/$B.json
python scripts/check_tracker_lr.py     --sync-root $WORK/ego_data_sync --datasets $B --profile $P --output $G/tracker_lr.$B.json
python scripts/check_startup_stance.py --sync-root $WORK/ego_data_sync --datasets $B --profile $P --output $G/startup_stance.$B.json
python scripts/check_glove_sync.py     --sync-root $WORK/ego_data_sync --datasets $B --output $G/glove_sync.$B.json
python scripts/check_hand_identity.py  --sync-root $WORK/ego_data_sync --datasets $B --output $G/hand_identity.$B.json
```

With several batches, concatenate the per-batch `tracker_lr.<batch>.json` files into one `tracker_lr.json`, and the same for `glove_sync`. They are plain JSON lists.

The drift check uses projection residuals, so it runs **after** projection:

```bash
python scripts/check_tracker_drift.py --sync-root $WORK/input/ego_data_sync \
  --projected $WORK/out/projected --output $G/tracker_drift.json
```

| Check | Script | How it decides |
|---|---|---|
| Tracker sides swapped | `check_tracker_lr.py` | The head mounting gives the wearer's left direction; the "left minus right" of each tracker pair, projected onto it, should stay positive throughout. Decided on the opening stance, because "the frames where the limbs are widest apart" is unreliable when the feet stay close together |
| Tracker drift or detachment | `check_tracker_drift.py` | Over minutes the body is rigid: the distance from head to wrist or ankle goes up and down with motion but does not keep growing or shrinking. Reports the linear trend of that distance, `trend_m`, and how much the tracker's residual grows from the first to the last fifth, `residual_growth_mm` |
| Glove pairing and timing | `check_glove_sync.py` | MANUS glove roots carry orientation, so their angular speed is cross-correlated with the wrist tracker's: the lag at the peak is the timing offset, and a stronger crossed pairing means the sides are swapped. Data is resampled to 30 Hz before differencing and angular speed is clipped at 20 rad/s to reject bad frames. Wuji glove roots carry no orientation and are marked `unsupported` |
| Glove handedness | `check_hand_identity.py` | Determines from the skeleton's own handedness, frame by frame, which hand each stream belongs to, which fixes `slot` directly |
| Opening stance | `check_startup_stance.py` | Whether the geometry of the first seconds makes sense; wrists metres apart, for example, mean drift |
| Fit residual | `tracker_diagnostics.npz` | A per-tracker median above 15 mm is high, above 60 mm abnormal; the overall 75th percentile is only 4–12 mm |

When reading the results:

- A tracker that never moves can look like a side swap (two fixed poses differ by a fixed vector); check that it moves first.
- The glove verdict is relative to the wrist trackers. If the wrist trackers themselves are swapped, the glove verdict flips: actual swap = glove judged swapped XOR wrist trackers judged swapped.

## 9.2 Fixes

```bash
python scripts/fix_tracker_lr.py --report $G/tracker_lr.json \
  --sync-root $WORK/ego_data_sync --output-root $WORK/ego_data_lr_fixed
python scripts/fix_glove_lr.py   --report $G/glove_sync.json --tracker-report $G/tracker_lr.json \
  --sync-root $WORK/ego_data_sync --output-root $WORK/ego_data_lr_fixed
```

- Both scripts accept `--dry-run` to list what they would change.
- Corrections go to a parallel folder `$WORK/ego_data_lr_fixed/<batch>/<recording>/`; **the raw data is never modified**. Unchanged files are linked, so a corrected copy works directly as input.
- The tracker fix swaps the left and right tracker streams and records `tracker_lr_correction` in the npz; the glove fix only rewrites `session.json`, swapping the two gloves' `slot` and recording the evidence in `glove_lr_correction`.
- After fixing, build the merged input as in 7.5; projection and hand fusion both read from it. If a recording was already projected, delete its results under `references/`, `proposals/` and `projected/` and run it again.

## 9.3 Three-pane review pages

Each recording gets a stand-alone `<slug>.review.html`. The `slug` is `session_<date>_<time>`, without the Chinese task name, because Chrome refuses media whose file name contains it; add a suffix if two collide. `run_pipeline.sh review` does all three steps below:

```bash
R=$WORK/out/review; mkdir -p $R
for npz in $WORK/out/unified_30hz/*.npz $WORK/out/unified_30hz_nobody/*.npz; do
  name=$(basename "$npz" .npz)
  slug=$(printf '%s' "$name" | sed -E 's/^(session_[0-9]+_[0-9]+).*/\1/')
  mp4=$(ls $WORK/ego_data_sync/*/"$name"/"$name".mp4 2>/dev/null | head -1)
  # 1. a 360p, 15 fps copy of the video for in-page playback
  [ -n "$mp4" ] && [ ! -f $R/$slug.360.mp4 ] && ffmpeg -y -v error -i "$mp4" -vf "scale=-2:360" -r 15 \
      -c:v libx264 -preset veryfast -crf 32 -an -movflags +faststart $R/$slug.360.mp4
  # 2. the page data (subsampled, compressed mesh)
  python scripts/build_review_payload.py --unified "$npz" --output-dir $R --body-model-path models/smplx --max-frames 300
  [ "$name" != "$slug" ] && mv -f $R/$name.payload.json $R/$slug.payload.json && mv -f $R/$name.mesh.i8c.gz $R/$slug.mesh.i8c.gz
  # 3. the page
  v=(); [ -f $R/$slug.360.mp4 ] && v=(--video)
  python scripts/make_review_html.py $R/$slug.payload.json --slug $slug "${v[@]}"
done
```

![Three-pane review page of one Wuji recording](../assets/img/review_page.jpg)

*The review page of a weighing recording processed with the toolkit on one machine. Top left, the GoPro video; top right, SMPL-X with fingers; below, the contact of 17 regions on each hand (yellow means contact) and a strip of contact over time. The header states the touch thresholds and the frame subsampling.*

All three panes come from the same 30 Hz export, so the video frame, mesh pose and touch reading at any slider position are the same instant by construction; the page does not have to align anything itself:

1. **Egocentric video** (the 360p copy);
2. **The SMPL-X body mesh** with the five trackers. Each frame has 10,475 vertices; by default at most 300 frames are taken at even intervals and stored compressed in the page;
3. **Hands and touch**: hand skeletons and the contact of the 17 regions, with contacting regions highlighted and named.

Recordings whose projection failed also get a page (from `unified_30hz_nobody/`) with video and touch only; the mesh pane states why it failed.

!!! note
    The page stores its data inside the HTML (`const P = …`), so editing the `.payload.json` next to it has no effect; regenerate the page instead. The mesh has at most 300 frames, so long recordings play at 0.6–2 Hz on the page; the stutter comes from subsampling, not from encoding. For full frame rate, read the `unified_30hz` npz directly.

## 9.4 Gallery

```bash
python scripts/build_ego_gallery.py --batch-root $WORK/out --output-dir $WORK/out/gallery \
  --sync-root $WORK/input/ego_data_sync --body-model-path models/smplx --device cpu --title "EgoExplorer review"
```

`gallery/index.html` is a single file that opens with a double-click:

- one card per recording, with skeleton projections and the five measured tracker points in three frames of fixed physical size (1.5 m wide, 2.25 m tall, fixed floor line), a residual curve, a thumbnail of the root path and statistics. The frames deliberately do not auto-scale, so problems such as a body sunk below the floor or a wrong scale are obvious at a glance;
- sortable by residual, rotation or displacement and filterable by any check; failed recordings are listed with their cause;
- each card links to its review page (in `../review/` by default).

## 9.5 What to check by eye

| Check | Normal | If not, look at |
|---|---|---|
| Body and video in sync | Turns, reaches and squats happen at the same time as in the video | Timestamp conversion, `video_time_offset_s` |
| Sides | An object picked up with the right hand in the video is in the mesh's right hand | Tracker or glove sides (9.1) |
| Feet | The supporting foot neither sinks, floats nor slides | Floor estimate, L3 weight, torso lean limit |
| Bending, squatting, sitting | The torso really leans forward instead of the whole body sinking | Whether the torso lean limit is still 15° |
| Fingers | Bend towards the palm, never backwards | Glove sides, mirroring of the left hand |
| Touch | Contact regions and times match where and when the hand touches objects in the video | Glove sides, glove health table |
| Residual curve | Steady, from a few millimetres to a centimetre or two | Strap slip, detached tracker, wrong calibration |

Record each verdict (pass, issue, re-record) per recording in one table; it is what you select release data from later.
