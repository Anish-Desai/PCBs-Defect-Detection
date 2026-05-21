# PCB Inspector

`PCB Inspector` is an automated defect inspection pipeline for printed circuit board (PCB) panel images. It processes a single full-panel image containing 10 PCB slots, detects and extracts each slot, then evaluates each slot through a three-stage AI inspection workflow.

---

## System Overview

The system architecture is composed of two primary components:

1. Slot extraction using a YOLO crop model
2. Slot-level defect inspection using a cascading YOLO/CNN/YOLO pipeline

### Stage 1 — Slot Extraction

A YOLO-based crop model detects each PCB slot in the panel image. Detected regions are cropped and normalized to portrait orientation before defect analysis.

### Stage 2 — Slot-Level Inspection

Each extracted slot is evaluated by the following pipeline:

```text
Panel image
    │
    ▼
[Crop model] → 10 independent slot images
    │
    ▼
[YOLO-1 initial scan] ─── defect detected ──→ DEFECTIVE
    │ no defect
    ▼
[CNN classifier] ─── all OK ───→ OK
    │ uncertain / suspicious
    ▼
[YOLO-2 recheck] ─── defect confirmed ──→ DEFECTIVE
    │ not confirmed
    ▼
OK_WITH_WARNING
```

| Stage  | Model             | Purpose                                                    |
| ------ | ----------------- | ---------------------------------------------------------- |
| YOLO-1 | `yolo_model.pt`   | Initial fast defect detection at confidence threshold 0.25 |
| CNN    | `cnn_model.keras` | Deep classification of slot crops and suspicious regions   |
| YOLO-2 | `yolo_model.pt`   | Low-threshold recheck at confidence threshold 0.15         |

### Output States

| Status            | Description                                                    |
| ----------------- | -------------------------------------------------------------- |
| `OK`              | No defects detected by YOLO-1, and CNN did not raise a warning |
| `DEFECTIVE`       | A defect was confirmed by YOLO-1 or by the YOLO-2 recheck      |
| `OK_WITH_WARNING` | CNN flagged a potential issue, but YOLO-2 did not confirm it   |

---

## Defect Taxonomy

### CNN classification labels (10)

| Label              | Description                                |
| ------------------ | ------------------------------------------ |
| `Corrected_BR`     | Normal corrected bridge region             |
| `Corrected_IC2`    | Normal corrected IC2 region                |
| `Corrected_R7`     | Normal corrected resistor R7 region        |
| `Corrected_iC1`    | Normal corrected IC1 region                |
| `defected_BR!`     | Defective bridge present                   |
| `miss_align_ic1`   | IC1 misalignment                           |
| `miss_aligned_ic2` | IC2 misalignment                           |
| `missing_R7`       | Missing resistor R7                        |
| `shifted_R7`       | Resistor R7 shifted from expected position |
| `white_R7`         | Resistor R7 discolored/white appearance    |

### YOLO defect detection labels (7)

| Label        | Description                     |
| ------------ | ------------------------------- |
| `correct`    | No defect detected              |
| `glue`       | Glue contamination present      |
| `missalign`  | Component misalignment detected |
| `missing`    | Missing component detected      |
| `r_shift`    | Component shift defect          |
| `shifted`    | Shifted component detected      |
| `upsidedown` | Component placed upside down    |

---

## Repository Layout

```
├── data/                        # Example PCB crop images
│   ├── pcb_bottom_left.jpg
│   ├── pcb_bottom_right.jpg
│   ├── pcb_center.jpg
│   ├── pcb_top_left.jpg
│   └── pcb_top_right.jpg
│
├── models/                      # External model artifacts (not tracked)
│   ├── cnn_model.keras          # CNN inference model
│   ├── crop_model.pt            # YOLO slot extraction model
│   └── yolo_model.pt            # YOLO defect detection model
│
├── src/
│   ├── config/
│   │   └── config.py            # Model path definitions, class maps, thresholds
│   │
│   ├── data/
│   │   └── preprocessing.py     # Image loading, crop normalization, slot saving
│   │
│   ├── pipeline/
│   │   └── analyzer.py          # Slot extraction and inspection pipeline logic
│   │
│   ├── utils/
│   │   └── helpers.py           # Label mapping and inference helpers
│   │
│   ├── web/
│   │   ├── annotate.py          # Bounding box rendering utilities
│   │   ├── app.py               # Flask web application and API endpoints
│   │   └── pdf_report.py        # PDF report assembly
│   │
│   ├── main.py                  # CLI execution entry point
│   └── requirements.txt         # Python dependency manifest
│
└── README.md
```

---

## Setup Instructions

### 1. Create and activate a Python virtual environment

```bash
python -m venv venv

# Windows
venv\Scripts\activate

# macOS / Linux
source venv/bin/activate
```

### 2. Install Python dependencies

```bash
pip install -r src/requirements.txt
```

### 3. Place model artifacts

Copy the trained models into the local `models/` folder:

```text
models/cnn_model.keras
models/crop_model.pt
models/yolo_model.pt
```

> Model files are not included in the repository and must be supplied separately.

---

## Execution

### Launch the web application

```bash
python -m src.web.app
```

Open the application at `http://localhost:5000`.

### Run the CLI pipeline

```bash
python -m src.main
```

This command executes the full panel analysis pipeline and prints the JSON result summary for the configured test image.

---

## Web Interface Features

- Upload a panel image containing 10 PCB slots
- Adjust YOLO confidence thresholds for initial scan and recheck stages
- View per-slot status cards with color-coded pass/fail indicators
- Inspect detailed slot diagnostics in a modal view
- Download individual slot PDF reports or a consolidated full-panel report

Status indicator semantics:

- `OK` — slot passed automated inspection
- `DEFECTIVE` — defect confirmed by the pipeline
- `OK_WITH_WARNING` — suspicious slot requiring manual review

---

## API Endpoints

### `POST /api/inspect`

Submit a panel image for automated inspection.

- Request: `multipart/form-data` with form field `image`
- Response: JSON array containing per-slot inference results

Example response payload:

```json
[
  {
    "slot_index": 1,
    "slot_label": "pcb",
    "confidence": 0.91,
    "report": {
      "status": "DEFECTIVE",
      "stage": "YOLO initial",
      "yolo_initial": [
        {
          "class_id": 3,
          "label": "missing",
          "confidence": 0.87,
          "bbox_xyxy": [120, 45, 310, 198]
        }
      ],
      "cnn": {},
      "yolo_recheck": []
    }
  }
]
```

### `GET /api/health`

Returns service health status:

```json
{ "status": "ok" }
```

---

## Dependency Summary

| Package                | Role                                               |
| ---------------------- | -------------------------------------------------- |
| `ultralytics`          | YOLO model inference and detection pipelines       |
| `tensorflow` / `keras` | CNN model loading and prediction                   |
| `keras-cv`             | Data augmentation and preprocessing layers         |
| `opencv-python`        | Image reading, cropping, and annotation operations |
| `pillow`               | Image format conversion and encoding               |
| `flask`                | Web application server and REST API layer          |
| `reportlab`            | PDF report generation                              |

---

## Implementation Notes

- Model binaries are excluded from version control.
- Virtual environments should not be committed; recreate with the provided setup instructions.
- The crop model operates at `conf=0.5` to match the original training and inference configuration.
- Default defect detection thresholds are `YOLO-1: 0.25` and `YOLO-2: 0.15`; these are adjustable through the UI.
