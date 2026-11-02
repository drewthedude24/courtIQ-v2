# Later Milestones And Pose/Fatigue Goal (reference; not setup tasks)

## 8. Later Application Milestones (Not Setup Tasks)

After basic detection is understood and assessed, propose only the next small step:
select/track the correct ball, identify attempts/releases, then assess outcomes.
Keep detection, identity tracking, release recognition, scoring, drawing, persistence,
and HTTP handling understandable and separately testable. No requirement to reuse
the old implementation or add its complexity all at once.

Use actual observations as evidence. Prediction can guide search through short gaps;
it cannot alone prove a basket. Support MAKE, MISS, and explicit UNCERTAIN/review
when scoring is introduced. Missing footage is not automatically a miss. Keep uncertain
attempts visible and distinguish automatic predictions from reviewed corrections.
Evaluate missed/false attempts, wrong-ball switches, outcome accuracy, and coverage
separately. Synthetic trajectories or plausible total counts do not establish accuracy.
Track coordinate spaces, timestamps, source FPS, and any analysis stride explicitly.

Intended full-stack direction, only when requested:
- React frontend, deliberately minimal; TypeScript is an option to agree on at setup.
- FastAPI Python backend wrapping the existing analysis function, not duplicating CV.
- PostgreSQL for session/shot metadata; do not add a database before there is data
  and a simple persistence requirement to understand.
- Original/processed videos stored as files initially; SQL stores their references.
- Upload -> job/status -> browser playback/timeline -> corrections -> saved session.
- Validate uploads, generate safe paths, isolate result directories, surface errors,
  and test browser seeking/codec compatibility. Do not block the API event loop
  with long model work or add Redis/Celery/microservices before a demonstrated need.
- Keep original prediction, reviewed result, time/evidence, and model/config versions
  separate. Do not silently duplicate shots on reanalysis or publish private videos.

## 9. Release Pose And Fatigue ML: Preserve The Future Goal

Once release detection exists, plan to save the selected shooter's release-time pose
and a short surrounding window, with source time/frame, landmark confidence,
coordinate convention/dimensions, and model/feature versions. Do not fabricate release
pose when only a rim approach was observed. Missing landmarks are missing, not zeros.

Possible descriptive features include elbow/knee angles and within-session changes.
Camera view and pose uncertainty limit interpretation. Fatigue modeling additionally
requires appropriate labeled data, such as consented exertion reports and a documented
collection protocol. Elapsed time, shot number, angle changes, or misses alone are
not fatigue ground truth. Use a simple baseline and held-out sessions/players before
making predictive claims. Avoid future-information leakage and medical/causal claims.
There is no trained fatigue model, pose feature store, or physiological score today.
