# ANTS — Smart Drone Traffic Analyzer

> A production-grade, full-stack application that processes aerial drone footage to detect, track, and count vehicles in real time — with a live progress feed, annotated video output, and an auto-generated Excel report.

---

## 📹 Demo

> ** https://drive.google.com/file/d/1kpuTNDOGSmUmqGgu0SFQJ2hHqvTyx55Z/view?usp=sharing


---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Tech Stack](#tech-stack)
3. [Architecture](#architecture)
4. [Frontend ↔ Backend Communication](#frontend--backend-communication)
5. [Computer Vision Pipeline](#computer-vision-pipeline)
6. [Tracking Methodology & Edge Cases](#tracking-methodology--edge-cases)
7. [Automated Reporting](#automated-reporting)
8. [Local Setup — Step by Step](#local-setup--step-by-step)
9. [Project Structure](#project-structure)
10. [Engineering Assumptions](#engineering-assumptions)
11. [Evaluation Criteria Mapping](#evaluation-criteria-mapping)

---

## Project Overview

The Smart Drone Traffic Analyzer accepts a standard `.mp4` drone video, runs a YOLOv8 computer vision pipeline to detect and track every vehicle across frames, and produces:

- A **processed video** with real-time bounding boxes and unique track IDs overlaid on every vehicle
- A **live progress feed** in the browser showing processing stage and frame count
- An **Excel report** with total unique vehicle counts, per-type breakdown, per-vehicle timestamps, and processing metadata

The application is a fully decoupled web system: a React frontend communicates with a Python FastAPI backend over REST and Server-Sent Events (SSE).

---

## Tech Stack

| Layer | Technology | Reason |
|---|---|---|
| Frontend | React 18 + Vite | Fast HMR dev server, lightweight bundle |
| Styling | Pure CSS with CSS variables | No framework dependency, full design control |
| HTTP client | Axios | Upload progress callbacks, clean interceptors |
| Live updates | Browser `EventSource` (SSE) | Push-based, no polling, works over HTTP/1.1 |
| Backend | FastAPI + Uvicorn | Async-native, automatic OpenAPI docs, fast |
| Background jobs | FastAPI `BackgroundTasks` | Non-blocking processing without a task queue |
| Computer vision | Ultralytics YOLOv8 | State-of-the-art detection, COCO pretrained |
| Video I/O | OpenCV (`opencv-python-headless`) | Frame-level read/write control |
| Tracking | Custom IoU tracker (ByteTrack-inspired) | No extra dependencies, full control over logic |
| Reporting | openpyxl | Native Excel with multi-sheet styled output |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        BROWSER  (React)                         │
│                                                                 │
│   ┌─────────────┐    ┌──────────────────┐    ┌──────────────┐  │
│   │ UploadZone  │───▶│  ProgressPanel   │───▶│ ResultsPanel │  │
│   │ drag & drop │    │  SSE live feed   │    │ video+report │  │
│   └─────────────┘    └──────────────────┘    └──────────────┘  │
└────────────┬──────────────────┬──────────────────┬─────────────┘
             │ POST /api/upload │ GET /stream (SSE) │ GET /report
             ▼                  ▼                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                      FASTAPI  BACKEND                           │
│                                                                 │
│  routes.py                                                      │
│  ├── POST /api/upload        → save file, register job,        │
│  │                             fire BackgroundTask             │
│  ├── GET  /api/job/:id/stream → SSE: push job JSON every 500ms │
│  ├── GET  /api/job/:id/report → stream Excel file download     │
│  └── GET  /api/job/:id/video  → stream annotated MP4           │
│                                                                 │
│  job_manager.py  (thread-safe in-memory job store)             │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  pipeline.py  (runs in background thread)                 │ │
│  │                                                           │ │
│  │  OpenCV VideoCapture                                      │ │
│  │       │                                                   │ │
│  │       ▼  every 2nd frame                                  │ │
│  │  YOLOv8 inference  (CUDA / MPS / CPU — auto-detected)    │ │
│  │       │                                                   │ │
│  │       ▼                                                   │ │
│  │  VehicleTracker.update()                                  │ │
│  │  ├── IoU matching                                         │ │
│  │  ├── Confirmation gate (MIN_HITS = 3)                     │ │
│  │  ├── Occlusion buffer  (MAX_AGE  = 30)                    │ │
│  │  └── Re-entry dedup   (centroid + time window)            │ │
│  │       │                                                   │ │
│  │       ▼                                                   │ │
│  │  Annotate frame → VideoWriter → output MP4                │ │
│  │       │                                                   │ │
│  │       ▼  (on completion)                                  │ │
│  │  reporter.py → 3-sheet Excel workbook                     │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## Frontend ↔ Backend Communication

### 1. Video Upload
The user selects or drops an `.mp4` file. The frontend sends it as `multipart/form-data` via Axios:

```
POST /api/upload
Content-Type: multipart/form-data

→ Response: { job_id, filename, status: "queued" }
```

Axios's `onUploadProgress` callback drives the upload progress bar in real time (0–100%).

### 2. Live Processing Updates (SSE)
Immediately after receiving `job_id`, the frontend opens a persistent SSE connection:

```
GET /api/job/:id/stream
Accept: text/event-stream
```

The backend pushes the full job JSON object every 500 ms while the pipeline runs:

```json
{
  "status": "processing",
  "progress": 47,
  "stage": "Processing frame 940/2000",
  "frames_processed": 940,
  "total_frames": 2000
}
```

When `status` becomes `"completed"` or `"failed"`, the frontend closes the connection automatically. This is push-based — no polling, no WebSocket upgrade needed.

### 3. Report Download
A plain `<a href="/api/job/:id/report" download>` anchor tag. FastAPI returns the `.xlsx` file via `FileResponse` with the correct `Content-Disposition` header. The browser handles the download natively.

### 4. Video Playback
A native HTML `<video src="/api/job/:id/video">` element. FastAPI streams the annotated MP4 via `FileResponse`. No base64 encoding or blob URLs needed.

### 5. Vite Dev Proxy
In development, Vite proxies all `/api` and `/outputs` requests from port `5173` → `8000`, eliminating CORS issues entirely. In production, CORS is configured explicitly on the FastAPI side.

---

## Computer Vision Pipeline

### Step 1 — Video Ingestion
OpenCV's `VideoCapture` opens the uploaded file and reads metadata: total frame count, FPS, and resolution. These drive the progress percentage and report output.

### Step 2 — Frame Sampling
Processing every single frame is computationally expensive and unnecessary. At typical drone video frame rates (24–30 fps), vehicles move only a few pixels between consecutive frames. The pipeline processes **every 2nd frame** (`FRAME_SKIP = 2`), halving inference time with negligible accuracy loss.

### Step 3 — YOLOv8 Detection
Each sampled frame is passed to a **YOLOv8-nano** model (`yolov8n.pt`), pre-trained on the COCO dataset. Only vehicle classes are retained:

| COCO Class ID | Label |
|---|---|
| 2 | car |
| 3 | motorcycle |
| 5 | bus |
| 7 | truck |

Detection parameters:
- **Confidence threshold**: `0.35` — low enough to catch partially visible vehicles, high enough to suppress background noise
- **Inference size**: `640px` — YOLOv8's native stride, optimal speed/accuracy tradeoff

### Step 4 — Device Selection (Auto)
The pipeline auto-detects the best available compute device at startup:

```python
if torch.cuda.is_available():    device = "cuda"   # NVIDIA GPU
elif torch.backends.mps.is_available(): device = "mps"  # Apple M1/M2/M3
else:                            device = "cpu"    # Universal fallback
```

The selected device is logged to the terminal and shown in the UI progress stage.

### Step 5 — Frame Annotation
Each confirmed track is drawn onto the frame with a color-coded bounding box and label (`#ID class confidence%`). A HUD counter showing total unique vehicles is overlaid in the top-left corner. The annotated frame is written to the output MP4.

---

## Tracking Methodology & Edge Cases

The tracker lives in `backend/app/services/tracker.py` and is a custom IoU-based multi-object tracker inspired by the core matching logic of ByteTrack. It was built from scratch to give full control over the counting and deduplication logic.

### How Tracks Are Matched

For every new frame, all current detections are matched against all existing tracks using **Intersection over Union (IoU)**:

1. An IoU matrix is computed for every (track, detection) pair.
2. Pairs are greedily assigned starting from the highest IoU value.
3. Any pair with IoU below `0.30` is rejected — they are too far apart to be the same object.
4. Unmatched detections become new **tentative** tracks.
5. Unmatched tracks have their age counter incremented.

### Edge Case 1 — Ghost Detections / Flickering
**Problem**: YOLO occasionally fires a spurious single-frame detection on a shadow, road marking, or background element. Counting these would inflate the total.

**Solution — Confirmation Gate**: A track must be matched successfully in at least `MIN_HITS = 3` consecutive frames before it is considered "confirmed" and eligible for counting. Single-frame ghosts never reach this threshold.

### Edge Case 2 — Temporary Occlusion (Lamppost, Tree, Another Vehicle)
**Problem**: A vehicle passing behind an obstacle disappears from detection for several frames. Without handling this, the tracker would delete the track and create a new one when the vehicle reappears — counting it twice.

**Solution — MAX_AGE Buffer**: Unmatched tracks are not immediately deleted. They are kept alive for up to `MAX_AGE = 30` frames (~1 second at 30 fps) without a match. If the vehicle reappears within this window, it is matched back to the same track ID. The vehicle is never double-counted because the track ID never changed.

### Edge Case 3 — Vehicle Slowing or Stopping
**Problem**: A slow-moving or stationary vehicle may have very little overlap between its predicted bounding box and its actual position across frames, causing the IoU matcher to drop it.

**Solution**: Because the IoU threshold (`0.30`) is deliberately permissive rather than strict, and because the `MAX_AGE` buffer gives substantial grace time, stationary vehicles maintain their track as long as the detector keeps seeing them.

### Edge Case 4 — Vehicle Re-entry After Leaving Frame
**Problem**: A vehicle exits the frame edge entirely, the track expires past `MAX_AGE` and is deleted. The same vehicle then re-enters the frame. A naive counter would count it again.

**Solution — Re-entry Deduplication**: When a confirmed track is deleted (age > `MAX_AGE`), it is moved to an `exited` list with its last known centroid and a timestamp. When a new track is first confirmed, it is checked against this list. If all three conditions are met simultaneously, it is treated as the same vehicle and **not counted again**:

- Same vehicle class
- Centroid within `REENTRY_DIST_THRESHOLD = 80` pixels of the exited track's last position
- Within `REENTRY_TIME_WINDOW = 5.0` seconds of the exit

### Counting Rule
A vehicle ID is added to the `counted_ids` set **exactly once** — at the moment its track first becomes confirmed AND passes the re-entry check. All subsequent frames for that track ID are ignored for counting purposes.

---

## Automated Reporting

The Excel report is generated by `backend/app/services/reporter.py` using `openpyxl`. It contains three styled sheets:

### Sheet 1 — Summary
| Field | Value |
|---|---|
| Report Generated | UTC timestamp |
| Job ID | UUID |
| Total Unique Vehicles | Integer |
| Per-type counts | car, truck, bus, motorcycle |
| Total Frames | Integer |
| Video FPS | Float |
| Video Duration | Seconds |
| Processing Duration | Seconds |

### Sheet 2 — Vehicle Log
One row per unique counted vehicle:

| Track ID | Vehicle Type | First Frame | First Timestamp (s) | Detection Confidence |
|---|---|---|---|---|
| 4 | Car | 12 | 0.400 | 0.87 |
| 7 | Truck | 34 | 1.133 | 0.91 |

### Sheet 3 — Type Breakdown
Vehicle types sorted by count descending, with a styled TOTAL row.

---

## Local Setup — Step by Step

### Prerequisites

| Tool | Version | Check |
|---|---|---|
| Python | 3.10+ | `python --version` |
| Node.js | 18+ | `node --version` |
| npm | 9+ | `npm --version` |
| Git | any | `git --version` |

> ⚠️ **PyTorch 2.6 note**: If you are using PyTorch 2.6+, make sure `ultralytics` is up to date (`pip install --upgrade ultralytics`). PyTorch 2.6 changed the default `weights_only` behavior in `torch.load`, and older ultralytics versions will crash on model load.

---

### Step 1 — Clone the Repository

```bash
git clone https://github.com/YOUR_USERNAME/drone-traffic-analyzer.git
cd drone-traffic-analyzer
```

---

### Step 2 — Backend Setup

```bash
cd backend

# Create a virtual environment
python -m venv venv

# Activate it
source venv/bin/activate          # Mac / Linux
venv\Scripts\activate             # Windows

# Install all Python dependencies
pip install -r requirements.txt

# If you are on PyTorch 2.6+ and see a weights_only error, also run:
pip install --upgrade ultralytics

# Create required directories
mkdir uploads outputs

# Start the FastAPI server
python -m app.main
```

Expected output:
```
INFO:     Uvicorn running on http://0.0.0.0:8000
INFO:     Application startup complete.
```

> `yolov8n.pt` (~6 MB) is downloaded automatically on first run from the Ultralytics CDN. An internet connection is required the first time only.

---

### Step 3 — Frontend Setup

Open a **second terminal** window (keep the backend running):

```bash
cd frontend

# Install Node dependencies (includes Vite, React, Axios)
npm install

# Start the dev server
npm run dev
```

Expected output:
```
  VITE v5.x.x  ready in Xms
  ➜  Local:   http://localhost:5173/
```

---

### Step 4 — Use the Application

1. Open **http://localhost:5173** in your browser
2. Drag and drop your `.mp4` drone video onto the upload zone, or click to browse
3. The upload progress bar fills as the file transfers to the backend
4. The processing pipeline starts automatically — watch live stage updates and frame counts
5. On completion:
   - The annotated video plays directly in the browser
   - Click **DOWNLOAD EXCEL REPORT** to save the `.xlsx` file

---

### Optional — API Explorer

FastAPI auto-generates interactive API documentation at:
```
http://localhost:8000/docs
```
You can test all endpoints directly from the browser without needing a client.



**Output video won't play in browser (black screen)**
```python
# In pipeline.py, change codec:
fourcc = cv2.VideoWriter_fourcc(*"mp4v")   # instead of avc1
```
*(Already auto-selected on Mac — only needed on Linux if `avc1` is unavailable)*

**Port 8000 already in use**
```bash
uvicorn app.main:app --port 8001
# Then update target in frontend/vite.config.js to http://localhost:8001
```

---

## Project Structure

```
drone-traffic-analyzer/
│
├── backend/
│   ├── requirements.txt
│   └── app/
│       ├── main.py                  # FastAPI app factory, CORS, static file mounts
│       ├── api/
│       │   └── routes.py            # All HTTP endpoints (upload, stream, report, video)
│       └── services/
│           ├── job_manager.py       # Thread-safe in-memory job state store
│           ├── pipeline.py          # Full CV pipeline: YOLO → tracker → writer → report
│           ├── tracker.py           # Custom IoU multi-object tracker
│           └── reporter.py          # 3-sheet styled Excel report generator
│
├── frontend/
│   ├── index.html
│   ├── vite.config.js               # Dev server + /api proxy config
│   ├── package.json
│   └── src/
│       ├── main.jsx                 # React DOM entry point
│       ├── App.jsx                  # Root component + 5-state app machine
│       ├── index.css                # CSS design tokens, global resets
│       ├── components/
│       │   ├── UploadZone.jsx       # Drag-and-drop upload with validation
│       │   ├── ProgressPanel.jsx    # SSE-driven live progress bar
│       │   └── ResultsPanel.jsx     # Stats cards, video player, download button
│       └── utils/
│           └── api.js               # Axios upload + SSE EventSource helpers
│
├── .gitignore
└── README.md
```

---

## Engineering Assumptions

The following decisions were made independently based on the requirements. Each is documented here as required by the assessment brief.

1. **In-memory job store (no database)**: `JobManager` uses a Python dictionary protected by a `threading.Lock`. This is sufficient for a single-process proof-of-concept. In production, this would be replaced with Redis or PostgreSQL to survive restarts and support multiple workers.

2. **FastAPI `BackgroundTasks` instead of Celery**: The assessment calls for a single-video processing flow. `BackgroundTasks` runs the pipeline in a background thread within the same process — no external broker required. The tradeoff is that concurrent uploads would queue behind each other. For a production system, Celery + Redis would be the right replacement.

3. **YOLOv8-nano for speed**: The nano model (`yolov8n.pt`, 3.2M parameters) provides the fastest inference with acceptable accuracy for standard vehicle classes. Engineers needing higher accuracy can change a single line in `pipeline.py`: `YOLO("yolov8s.pt")` or `YOLO("yolov8m.pt")`. No other code changes are required.

4. **Frame skip = 2**: Drone footage typically captures slow-moving traffic at 24–30 fps. Processing every other frame saves ~50% of compute time. Vehicles moving at typical road speeds shift by only a few pixels per frame at drone altitude, so this has negligible impact on tracking accuracy. For fast-moving scenes, set `FRAME_SKIP = 1` in `pipeline.py`.

5. **Codec auto-selection**: `avc1` (H.264) is used on Windows/Linux for the widest browser compatibility. On macOS (`platform.system() == "Darwin"`), `mp4v` is used automatically because OpenCV's `avc1` support on Mac depends on system codec availability and is unreliable.

6. **Re-entry window calibrated for medium-altitude drone footage**: The `REENTRY_DIST_THRESHOLD = 80px` and `REENTRY_TIME_WINDOW = 5.0s` constants were chosen for typical drone footage where vehicles span 30–80 pixels. For high-altitude footage (smaller vehicles), decrease the pixel threshold. For slow-moving traffic or long occlusions, increase the time window. Both constants are at the top of `tracker.py` for easy tuning.

7. **No authentication or file size hard limit**: The API accepts any video file within memory constraints. In production, this would require authentication, per-user quotas, and file size validation at the infrastructure level (e.g., nginx `client_max_body_size`).

8. **SSE over WebSockets for progress**: Server-Sent Events were chosen over WebSockets because progress updates are strictly server-to-client (unidirectional). SSE is simpler to implement, works over HTTP/1.1, and requires no special server upgrade. The browser `EventSource` API handles reconnection automatically.


