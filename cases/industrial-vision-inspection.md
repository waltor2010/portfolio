# Case Study: Industrial Surface-Defect Inspection

A surface-defect inspection system I built and shipped for a production line. It took over manual visual checking, and the second version was reworked so new product lines could be added by configuration instead of a from-scratch rebuild.

> 🌐 中文版 / Chinese version: [industrial-vision-inspection.zh.md](industrial-vision-inspection.zh.md)

---

## Background

The line was checking part surfaces by hand. As volume and part variety went up, results got inconsistent and defects started slipping through. There was no record of what had been checked or what failed.

Later it became clear the same team would serve several product lines and customers. Each new line had meant basically rebuilding the system. The second version came out of that.

---

## Problem

- Checking by hand was inconsistent and left no audit trail.
- No structured record of checks or failures.
- Know-how lived in operators' heads.
- Every new product line triggered a large, risky re-implementation.

---

## Constraints

| Constraint | Detail |
|-----------|--------|
| Latency | Inference had to keep pace with the line, camera by camera, in real time |
| Environment | Shop floor: dust, vibration, changing light |
| Offline | Some stations had no constant connection; a few were air-gapped |
| Maintenance | Operators run it daily; engineers are rarely on site |
| Variety | Many lines, each with different cameras, stations, and roles |

---

## Solution

Layered backend, a Vue frontend, and a PySide6 desktop shell for the floor:

```
┌──────────┐   MVS SDK  ┌──────────────┐   ┌────────────┐   ┌──────────────┐
│ Cameras  │──────────►│ Imaging /    │──►│ Model      │──►│ FastAPI +    │
│ (Hikvision│           │ Preprocess   │   │ (Anomalib /│   │ Vue3 dash.   │
│  line/area)│          │ (OpenCV)     │   │  YOLO)     │   │ + PySide6    │
└──────────┘           └──────────────┘   └────────────┘   │  desktop shell│
                                                           └──────┬───────┘
                                                       results API │ SSE
                                                           ┌──────▼───────┐
                                                           │ Review queue │
                                                           │ + audit log  │
                                                           └──────────────┘
```

- **Capture**: Hikvision MVS SDK drives line/area cameras, acquired per role (top / bottom / axis).
- **Pre-process**: OpenCV pipeline — resize, lighting normalization, ROI/mask.
- **Model**: Anomalib PatchCore for unsupervised detection (needs almost no labeled defects) and YOLO for supervised (planned).
- **Backend**: FastAPI, split into routes / services / schemas / repositories / models. Exposes a clean API: `POST /api/detection/start`, `GET /web`, plus camera / model / recipes / system_settings / users modules.
- **Frontend**: Vue3 dashboard for live status, defect counts, review queue.
- **Desktop shell**: PySide6 (QWebEngineView) hosts the same Vue UI for shop-floor control and offline use.
- **Audit**: every result logged with image hash, timestamp, and model version.

### The second version: configurable by design

The first version hardcoded each product line. The rework builds systems from a few basic dimensions:

```
Camera = imaging type (line/area) × brand/model (Hikvision…) × instance
Stage   = protocol (Modbus RTU) × model (register map + duties) × instance
Role (inspected face) = a business concept, named freely (axis/top/bottom/side/…)
Flow    = a configured sequence (feed / pose / capture / result / changeover)
```

Adding a product line now means writing a config (a camera spec, a stage spec, a role set). No code changes.

If I were pitching this to a new client, this is the part I'd lead with: it shows the system scales across lines without a rewrite.

---

## Key Technical Decisions

### 1. Anomalib PatchCore over a fully supervised model
We didn't have a large labeled defect set, and building one would have been slow and expensive. PatchCore detects "not normal" with almost no labels, so we could ship and then adapt per line.

### 2. PySide6 shell instead of browser-only on the floor
The floor needs offline operation and tight, low-latency device control. A desktop shell wrapping the web UI gave us that without a browser or IT dependency.

### 3. Configuration over per-line code
Cloning and modifying the system per line was the old way. Driving composition from data means a config change instead of a two-week rebuild. That protects margin and reliability across customers.

### 4. Layered FastAPI
Keeping logic out of the routes (routes / services / repositories) made the model and pipeline swappable and gave the frontend and integrators a clean API contract (OpenAPI).

---

## What I learned

The model was never the hard part. Lighting, triggering, and how cameras and stations fit together were. And the results only matter if the people on the floor trust them. Designing for configuration, not code, is what made the system usable across customers.

---

## Tags

`#computer-vision` `#fastapi` `#vue3` `#pyside6` `#anomalib` `#industrial` `#edge-ai`
