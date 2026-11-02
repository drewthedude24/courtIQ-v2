# New YOLO Model Workstream (reference; only when requested)

## 7. New YOLO Model: Separate, Reproducible Workstream

Training a better basketball detector is wanted. Preserve models/best.pt as the
immutable v1 baseline; never let a training output overwrite or silently replace it.
A fresh code repository does NOT mean training neural-network weights from scratch.
Explain the choice between fine-tuning a general pretrained detector and continuing
from the old basketball checkpoint; record initialization for each candidate.

After baseline inference is working, training can progress alongside the small
application milestones. Do not wait for a complete full-stack app, but do not
launch training merely because this goal is written here.

For each requested experiment:
1. Define the observed detection failure and success criteria before changing models.
2. Audit existing annotations and representative footage. Check missing balls,
   inconsistent boxes, tiny/blurred targets, and what the hoop box actually includes.
3. Curate permissioned examples across lighting, backgrounds, ball sizes, orientations,
   hands, rim/net occlusion, held balls, passes, and distracting background balls.
   Label all recognizable target-class instances in selected frames; include hard
   negative frames. Do not infer invisible ball locations as visual ground truth.
4. Split by recording/session/scene BEFORE extracting adjacent frames or augmenting.
   Keep train, validation, and held-out test independent; deduplicate across sources.
   Reused clips already tuned against are development footage, not unbiased final tests.
5. Version dataset/config/class mapping. Keep class IDs compatible with the baseline,
   or explicitly map any changed labels. A dedicated rim class/keypoints would require
   suitable new labels, not just a renamed hoop box.
6. Agree on model size, training imgsz, initialization, epochs, hardware, and time/cost.
   Start with one bounded experiment, not a large search or an assumed latest model.
   A larger model or resolution may help but must earn its extra compute.
7. Save each run under a unique training/runs/<experiment_id>/ directory (or another
   agreed artifact location). Preserve args, seed, dataset version/split, package
   versions, metrics, curves, checkpoints, and notes. Do not enable overwrite/reuse
   of an existing run directory casually. Never commit credentials or cloud tokens.
8. Evaluate v1 and candidates on the SAME held-out images and clips. Compare per-class
   precision, recall, mAP50/mAP50-95, relevant tiny/occluded-ball failures, and end-to-end
   latency on the intended hardware. Aggregate mAP can conceal weak basketball recall.
9. Use fixed inference settings for a controlled comparison; if tuning thresholds,
   do it separately on validation data for each model, then freeze for test evaluation.
   Report validation thresholds/settings explicitly, not just playback confidence.
10. Inspect visual failures. Promote a separately named candidate only after measured
    improvement at an acceptable false-positive/speed tradeoff and Andrew's agreement.
    Keep v1 and its results available even if a candidate wins.

Do not claim a new model is better because training loss falls, it looks good on
training clips, or the architecture is newer. Detector mAP is not make/miss accuracy.
Freeze model/config while comparing application logic so changes are attributable.
Training one better detector will not automatically solve ball identity or occlusion.
Check official APIs against the installed version before writing training commands:
https://docs.ultralytics.com/modes/train/
https://docs.ultralytics.com/modes/val/
