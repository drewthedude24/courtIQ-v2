> HISTORICAL: describes the OLD CourtIQ repo only. Not this repo's design.

# Legacy CourtIQ Technical And Product Handoff

> HISTORICAL REFERENCE: Andrew subsequently chose an entirely new repository and
> a from-scratch application implementation. The root CLAUDE.md contains the current
> direction, including preserving the old detector and training separate candidates.
> This document describes the OLD checkout only. Its pipeline, commands, branch,
> environment, and incremental-rebuild roadmap are not the new repo's starting state.
> Do not copy old application code, create the new repo, migrate assets, or launch
> training merely because those tasks appear in this historical handoff.

Inspected: 2026-11-01. All descriptions of existing code
refer to the working tree inspected on that date unless explicitly marked otherwise.
Recheck the branch, files, and settings before relying on this snapshot.

## 1. Project Motivation And Revised Scope

Andrew wants a basketball training product and a project through which he genuinely
learns Python, computer vision, FastAPI, React, relational databases, and eventually
ML on the data collected. He manually trained the basketball detection model but
does not yet fully understand several later AI-assisted tracking/API/UI components.
He wants understandable incremental work, not wholesale replacement or buzzwords.

The earlier ambition was live phone/webcam analysis, robust tracking across camera
angles/backgrounds, automatic release and make/miss detection, and eventual fatigue
analysis. The revised V1 preserves the purpose but reduces simultaneous unknowns:

```text
Upload recorded practice video
  -> process with versioned vision pipeline
  -> show replay and predicted shot timeline/counts
  -> review uncertain events and correct mistakes
  -> persist session and per-shot records
```

Use one target shooter and hoop, a stable camera, and adequate visibility initially.
Do not require a constant stream of newly recorded personal footage: reuse existing
clips and build a small permissioned evaluation set. Broad generalization is a later
measured expansion, not something to claim after tuning a handful of clips.

Optional shooter/hoop selection, uncertain-result review, database-backed corrections,
and a connected frontend are PRODUCT PLANS, not implemented features in this checkout.
Single-camera occlusion and 2D projection can leave outcomes genuinely ambiguous.
Do not convert ambiguity into confident labels merely to produce a complete count.

## 2. Repository Reality And Branch Differences

- Inspected branch: `old_ball_tracker`, tracking `origin/old_ball_tracker` locally.
- Modified tracked files included `vision/yolo_detector.py`, `vision/pose_detector.py`,
  and `vision/shot_detector.py`. Several important vision files and tests were untracked.
- Local `main` contains a different implementation and additional historical runners,
  including `clean_offline_runner.py` and `stitched_offline_runner.py`, not present in
  this working tree. A fresh clone/branch switch will not reproduce all local files.
- Some recent outputs ending in `_main` were produced using a temporary extraction
  of local `main`, not the current working tree. They are not current-code benchmarks.
- `output6/`, `output7/`, `output8/`, `output9/`, and `output10/` contained only
  `.DS_Store` files at inspection. Their names cannot establish historical settings.
- Root `backend/` and `docs/` were empty before these handoff documents were added;
  no root frontend directory, package manifest, or Python dependency manifest was found.
- `temp/frontend/` contains `dist/` and `node_modules/`; `temp/backend/` contains
  Python cache files but no `.py` sources. Do not reverse-engineer compiled caches
  as the default integration strategy or describe this as a maintained full-stack app.
- `temp/courtiq_temp.db` exists, with legacy `sessions` and `shots` tables. Preserve
  it; an old database does not mean current routes, auth, jobs, or source exist.
- README reflects the original webcam-first milestones, including outdated statements
  that training and offline replay have not happened. Reconcile it when requested.

## 3. Existing Models And Training Assets

Primary object checkpoint:

```text
runs3/basketball_yolo3/weights/best.pt
SHA-256: 0de10706b00dafdf21a20bd4e317cc3b74ce34602d33406691a3cfe705d41bcf
```

Preserve the entire associated training run, including `args.yaml`, `results.csv`,
and any other weights/results. Root `yolo26n.pt` is the original base model asset;
root `yolo26n-pose.pt` is the separate pose model. They are not interchangeable.

Recorded object-training settings in `runs3/basketball_yolo3/args.yaml`:

```text
Base model: yolo26n.pt
Epochs: 50
Batch: 32
Training imgsz: 512
Device: mps
Dataset copy: data/basketball_dataset3_yolo/
Classes: basketball, hoop (IDs 0, 1)
```

Training `imgsz` and inference `imgsz` are different settings. Enlarging inference
input does not retrain weights or guarantee recovery of a blurred/occluded ball.
Training configuration contains machine-specific absolute paths, including an old
Downloads dataset path. Check paths before attempting reproducible training.

Dataset metadata identifies Roboflow `basketball-qus2n/basketball-detection`, version 1,
license CC BY 4.0:
https://universe.roboflow.com/basketball-qus2n/basketball-detection/dataset/1
Verify attribution and model/dependency licensing before public/commercial release.
Detection validation metrics are not shot-event accuracy, fatigue accuracy, or proof
of generalization to unrelated footage. Do not invent such equivalences.

## 4. Current Offline Data Flow

`vision/offline_runner.py` creates fresh detector, pose, tracker, and shot-state
instances inside `process_video(input_path, output_path)` for each invocation.

1. Open the original video and validate FPS/dimensions and output writer.
2. Fit each source frame into orientation-aware inference bounds of `(1280, 720)`.
3. Request predicted-ball and rim focus regions in INFERENCE coordinates.
4. Run global YOLO periodically or when tracking is lost, plus relevant focus crops.
5. Run pose on the raw inference frame, not detector annotations.
6. Update ball identity/motion using detections, people, active-shot status, and rim region.
7. Update possession/release, then the independent shot-result state machine.
8. Emit result events, update counts, and reset relevant shot/ball state after results.
9. Scale geometry to source coordinates, draw overlays, and write original-size output.
10. Write per-frame JSONL, per-result JSONL, and run configuration; release resources.

Current detector defaults: full-frame `imgsz=960`, crop `imgsz=640`, `conf=0.12`,
full-frame interval `3`, detector device `mps`. With no usable focus region, the
detector falls back to global search. A strong existing ball detection can suppress
redundant crop inference. These behaviors affect actual model-call counts.

IMPORTANT: crops currently come from the RESIZED inference frame. Source-resolution
cropping to preserve more rim detail is an improvement proposal, not existing behavior.
Do not implement it without handling coordinate transformations and deduplication.

Primary output defaults:

```text
outputs/<input_stem>/
  annotated_improved.mp4
  configuration.json
  frames.jsonl
  shot_events.jsonl
```

`--output` changes the video path; sidecars use that video's parent directory. Reusing
a parent directory overwrites fixed-name sidecars. Use a fresh directory per experiment.
The function returns `(output_path, shot_events)`, not a database session object.

JSONL is newline-delimited JSON, not SQL or a database. Each `frames.jsonl` record
contains source frame index/time, detections, ball state/history, shot state/result,
counts, and inference diagnostics. Coordinates in those records are generally
inference-space; source dimensions and actual inference size are in configuration.
The drawn output instead uses scaled source-space geometry. Do not mix these spaces.

Events contain shot ID, result-frame time/index, result and reason, release/result
frame indices, and running counts. Event `timestamp_s` is the result time, NOT the
release time. The current runner does not save a release-time pose snapshot.

The writer uses `mp4v`, source dimensions, and source FPS. Output retains full frame
dimensions but analysis does not magically regain detail discarded during resizing.
Browser compatibility needs testing; H.264/yuv420p with suitable MP4 metadata is a
reasonable later encoding option, not a verified current dependency or automatic fix.
Do not assume FFmpeg is installed or available on PATH.

## 5. Component Responsibilities And Limitations

### Detection And Pose

`YoloDetector.detect_frame_with_fps()` returns annotated frame, merged detection
dictionaries, and detector-call FPS. Detections include class ID, confidence, bbox,
center, and `full`/`focus` source. Crop coordinates are mapped back before merging.
Its annotation work can contribute overhead even where the runner redraws output.

`PoseDetector.detect()` uses Ultralytics `model.track(persist=True, tracker="bytetrack.yaml")`
at `imgsz=384`, confidence `0.7`, CPU by default. It returns person IDs/bboxes/heights
and selected landmarks with confidence: right ear, shoulders, elbows, wrists, hips,
and knees. It currently does not expose every model landmark, such as ankles.
Pose tracks people; the custom basketball tracker is a separate implementation.

### Ball Tracking

`PredictiveBallTracker` currently uses state `[x, y, vx, vy, ax, ay]` with one update
treated as one video frame. Defaults include initialization threshold `0.32`, weak
candidate threshold `0.12`, prediction horizon `12`, and missing-frame limit `30`.
It requires wrist proximity to initialize, suppresses stationary background candidates,
supports handoffs and restricted rim reacquisition, and emits `DETECTED`/`PREDICTED`/`LOST`.

This is a SINGLE active-ball tracker, not a complete multi-ball identity system.
Wrist association is not yet explicit user-selected shooter identity. Held balls,
passes, multiple people, and ball crossings still require regression evaluation.
The local-main tracker differs, including a four-state constant-velocity filter.
Do not describe those two implementations as the same model or experiment.

### Shot Recognition And Outcomes

`ShotDetector` uses `IDLE`, `POSSESSION`, `POSSIBLE_SHOT`, and `RELEASED`. Release
depends on wrist separation, upward velocity, and height relative to the right ear,
with some height-normalized and confidence-aware checks. `RELEASED` persists until
completion. It still has frame-based cooldown/missing-person thresholds and view bias.

`ShotResultDetector` starts from release or a rim-approach fallback, smooths detected
hoop boxes, and judges observed trajectories near an estimated scoring corridor.
Rim center is estimated as `x1 + 0.35*w`, `y1 + 0.43*h` inside the hoop/net bbox.
This is a heuristic, not calibrated physical rim localization or a learned classifier.

Existing risks requiring tests, not automatic rewrites:

- A 2D crossing can pass in front of/behind the rim rather than through it.
- Above/below evidence can finalize a make before later bounce evidence is considered.
- A provisional crossing can mature after a delay without new confirming observations.
- Timeout after approach can become a miss even if the deciding interaction was hidden.
- Stale hoop history, camera cuts, and frame-based timing can invalidate assumptions.
- A rim-fallback start stores its start as `release_frame`; that is not a measured release.
- Cancelled attempts can leave `ShotDetector` released until another completion path.
- Adding uncertainty requires coordinated event, counter, overlay, API, and storage changes.

### Quick And Webcam Runners

`offline_runner_quick.py` preserves source output frames/FPS but analyzes every second
frame (`ANALYSIS_FRAME_STRIDE=2`), caches pose, and reduces some rim/drawing work.
Other defaults: full `960`, focus `640`, global interval `3`, pose interval `3`, rim
focus interval `2`, confidence `0.12`. Its logs use `_quick` filenames. Equal playback
FPS does not mean equal analysis rate or accuracy. Check time-step semantics before
changing stride. Its input is hardcoded separately near the bottom of the file.

`vision/main.py` is an older webcam script: resizes to `640x360`, uses the simpler
`BallTracker`, and passes a drawn-on YOLO frame to pose. It does not wire in the newer
result detector/offline logging. Do not claim live/offline feature or speed parity.

## 6. Validation And Benchmarking

Verified at inspection:

```sh
.venv/bin/python -m unittest discover -s tests -v
```

Seven tests passed: four tracker-selection/reacquisition tests and three synthetic
rim-trajectory outcome tests. Pytest was not installed. This validates specific
programmed behaviors, not empirical detection recall or real-video shot accuracy.

Build a small event-level benchmark before further broad tuning. Annotate actual
attempts, release where visible, result time, outcome, visibility, and relevant failure
conditions. Count false positives, missed attempts, wrong identities, and uncertainties.
Match predicted events to actual attempts; aggregate makes/misses alone is insufficient.
Evaluate outcome accuracy WITH coverage, including missed/uncertain attempts. Keep
clips from the same session and their derived frames in the same data split.

For every comparison record checkpoint/version, code revision or dirty-state snapshot,
exact config, hardware/device, input resolution/FPS/duration, end-to-end runtime,
event metrics, and representative failures. Do not quote earlier YouTube run counts
as hand-verified accuracy. Do not tune thresholds specifically to one troublesome shot.

## 7. Proposed Full-Stack Shape (Not Implemented)

Use a minimal React UI and FastAPI backend, then a relational store (eventually
PostgreSQL). Keep one Python vision implementation behind a small adapter. Start
with a local, single-user flow; add services only when justified by requirements.

```text
React: select/upload -> processing status -> video + shot timeline -> correction
FastAPI: validate upload -> create session/job -> run analysis -> expose results/media
SQL: session metadata + shot records + prediction/correction provenance
Files: original videos, processed videos, optional replay clips, diagnostic JSONL
```

A possible API, to agree on when implementing: `POST /sessions` for upload,
`GET /sessions/{id}` for state/results, media GET endpoints with browser seeking,
and `PATCH /sessions/{id}/shots/{shot_id}` for reviewed corrections. These are design
suggestions, not existing routes. A long analysis should not block the API event loop.
A simple local worker/process is sufficient to explore first; in-process jobs are
not durable across restarts. Do not add Redis/Celery/microservices preemptively.

Suggested records, not an existing migration:

- Session: ID, original name, input/output references, status, timestamps, error,
  checkpoint/config/code identifiers, source metadata, and processing duration.
- Shot: stable ID, session ID, attempt number, release/result times and evidence,
  predicted outcome, reviewed outcome, review state, reason, and optional pose reference.
- Keep original prediction distinct from corrected truth. Recompute counts from the
  chosen effective-result policy. Reanalysis must not silently duplicate events.

Store metadata/paths in SQL; large video blobs remain in local storage initially or
object storage later. Use generated paths instead of trusting uploaded filenames,
check size/type/decodability, bound processing, and do not serve arbitrary filesystem
paths. Authentication/access control becomes required before private videos are
exposed to multiple users. Never assume a public demo authorizes sharing all footage.

Legacy SQLite schema has `sessions(id, original_filename, input_path, output_path,
status, makes, misses, created_at, completed_at, error_message)` and
`shots(id, session_id, shot_number, result, timestamp_s, reason)`. It does not already
contain pose features, uncertainty, model versions, or prediction/correction history.

## 8. Release-Time Pose And Later Fatigue Research

Prepare the data carefully when implementing this requested future feature:

- Capture the selected shooter's pose at a confirmed release transition, ideally
  with a short surrounding window so a single noisy frame is not the only evidence.
- Save source time/index, shooter ID, original and inference dimensions, coordinate
  convention, raw keypoint positions/confidence, and pose/detector/config versions.
- Treat unavailable/occluded landmarks as missing, not `(0, 0)` measurements.
- Store derived features such as elbow/knee angles with units, confidence/validity,
  feature version, and the actual landmarks used. Preserve raw data to recompute them.
- Normalize for scale where appropriate, but record view/camera conditions; 2D angles
  are view-dependent and not automatically clinical or true 3D measurements.
- Do not fabricate a release pose for a shot recovered only near the rim. Mark
  release timing/pose unavailable or estimated, with provenance.
- Initially study within-session changes descriptively. Later collect appropriate
  fatigue/exertion labels, such as consented self-reported exertion and protocol data.
  Shot number, elapsed time, a miss, or an angle change alone is not fatigue ground truth.
- Establish a simple supervised baseline only after sufficient labeled examples.
  Evaluate on held-out sessions/players as appropriate, avoid future-frame/label leakage,
  and compare to trivial baselines. Do not promise fatigue prediction from sparse data.
- Position results as exploratory training insights, not diagnosis, injury prediction,
  causal coaching, or a physiological measurement without supporting validation.

## 9. Practical Order Of Work

1. Inventory current code and preserve the trained baseline without resetting anything.
2. Establish a fixed evaluation set and label the failure modes.
3. Improve the largest measured failure: detection, ball identity, release, or outcome.
4. Introduce explicit uncertainty/review while keeping attempt counts and evidence honest.
5. Expose analysis through a small, documented Python adapter and then FastAPI.
6. Add minimal React upload/status/playback, persistence, and reviewed corrections.
7. Add release-pose records and descriptive form comparisons; optimize measured bottlenecks.
8. Research fatigue ML and expand scene support only after the foundations are validated.

This is guidance, not authorization to implement every stage. Respect Andrew's current
request, prefer one understandable change at a time, and keep the learning process visible.

## 10. Environment Snapshot And References

Existing `.venv` at inspection: Python 3.14.0, Ultralytics 8.4.126, PyTorch 2.13.0,
OpenCV package 5.0.0.93, NumPy 2.5.2, FastAPI 0.141.1, Uvicorn 0.52.4,
python-multipart 0.0.32. SQLAlchemy and pytest were not installed. These are observed
versions, not a tested portable lockfile or a recommendation to install latest versions.
No current app startup scripts, dependency manifest, or deployment were verified.

Use official documentation when implementing unfamiliar APIs:

- Claude Code project context: https://code.claude.com/docs/en/memory
- FastAPI uploads: https://fastapi.tiangolo.com/tutorial/request-files/
- FastAPI SQL tutorial: https://fastapi.tiangolo.com/tutorial/sql-databases/
- React learning guide: https://react.dev/learn
- PostgreSQL documentation: https://www.postgresql.org/docs/
- Ultralytics documentation: https://docs.ultralytics.com/

Review alternative detectors/temporal models only when benchmarks identify a need.
YOLO is a baseline worth preserving, not an immutable requirement. A larger `imgsz`,
a different architecture, or a more elaborate tracker must earn adoption through
held-out accuracy, latency, maintainability, and licensing checks.
