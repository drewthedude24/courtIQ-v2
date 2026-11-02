# YOLO v1 (basketball + hoop): Preserved Baseline

Copied unchanged from the legacy repo on 2026-11-01. Do not modify.

- Legacy source: `runs3/basketball_yolo3/` in `/Users/andrewmeng/Desktop/courtIQ`
- `best.pt` SHA-256: `0de10706b00dafdf21a20bd4e317cc3b74ce34602d33406691a3cfe705d41bcf`
- Base model: `yolo26n.pt`; 50 epochs, batch 32, imgsz 512, device mps
- Classes: 0 = basketball, 1 = hoop
- Dataset: Roboflow `basketball-qus2n/basketball-detection`, version 1, CC BY 4.0
  https://universe.roboflow.com/basketball-qus2n/basketball-detection/dataset/1
- Files: `best.pt`, `last.pt`, `args.yaml`, `results.csv`, `data.yaml`, result plots.
- The dataset images are NOT in this repo (kept in the legacy archive, ~40 MB).
- `args.yaml` and `data.yaml` contain machine-specific/relative paths from the old
  Mac; they are provenance, not runnable configs here.
- Historical inference settings: imgsz 960, conf 0.12. Validation metrics here are
  box-detection metrics, not shot-event accuracy or proof of generalization.
- Known limits: small/blurred balls, rim/net occlusion, held balls, background balls.
