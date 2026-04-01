# Project Plan: Real-Time Vehicle Detection and Traffic Monitoring System

## File ne husi AI ne'ebe hau copy hela par sambil sai referensi ba hau nia claude Ai

**Prepared by:** [Your Name]
**Course:** Software Engineering
**Review Date:** [Date]

---

## 1. Problem Statement

Dili, Timor-Leste has no automated traffic monitoring system. Traffic flow is unmanaged and there is no data on vehicle counts, peak hours, or road congestion. This project proposes a software system that uses an AI model to automatically detect and count vehicles from video footage and display the results on a live web dashboard.

---

## 2. Proposed Solution

A web-based traffic monitoring application that:

- Plays a pre-recorded(for development testing) or live-feed traffic video through an AI detection model
- Draws bounding boxes around detected vehicles in real time
- Counts each vehicle type exactly once using a virtual counting line
- Saves all detections to a database
- Displays the annotated video feed and live counts on a dashboard

---

## 3. Requirements

### Functional Requirements


| #   | Requirement                                                                               |
| --- | ----------------------------------------------------------------------------------------- |
| FR1 | The system shall accept a pre-recorded or Live feed traffic video as input                |
| FR2 | The system shall detect vehicles in the video using a trained AI model                    |
| FR3 | The system shall draw bounding boxes and class labels on detected vehicles                |
| FR4 | The system shall count each vehicle exactly once when it crosses a virtual line           |
| FR5 | The system shall display the annotated video feed live on a web dashboard                 |
| FR6 | The system shall show live vehicle counts per category (car, truck, motorbike, bus)       |
| FR7 | The system shall save every detection to a database with class, confidence, and timestamp |
| FR8 | The system shall display a recent detection log on the dashboard                          |


### Non-Functional Requirements


| #    | Requirement                                             |
| ---- | ------------------------------------------------------- |
| NFR1 | The video feed must be smooth with minimal lag          |
| NFR2 | Database writes must not slow down the video processing |
| NFR3 | The system must run on a standard laptop without a GPU  |


---

## 4. Tech Stack


| Layer                   | Technology                | Reason                                                   |
| ----------------------- | ------------------------- | -------------------------------------------------------- |
| Web Framework           | Django 4.x                | Familiar, handles routing, ORM, and file uploads         |
| Real-time Communication | Django Channels           | Adds WebSocket support to Django                         |
| ASGI Server             | Daphne                    | Required to run Django Channels (HTTP + WebSocket)       |
| Channel Layer           | InMemoryChannelLayer      | No Redis needed for local single-user app                |
| AI Inference            | ONNX Runtime              | Lightweight (~15MB), fast on CPU, no PyTorch needed      |
| AI Model                | YOLOv8 Nano (.onnx)       | Custom trained, already exported to ONNX format          |
| Video Processing        | OpenCV (cv2)              | Reads video frames and draws bounding boxes              |
| Frame Data              | NumPy                     | Handles frame pixel arrays for pre-processing            |
| Database                | SQLite                    | Zero config, built into Django, sufficient for local use |
| Frontend Styling        | Tailwind CSS              | Clean utility-based styling, no custom CSS needed        |
| Frontend Rendering      | HTML5 Canvas + Vanilla JS | Smooth frame rendering, faster than swapping img tags    |


---

## 5. System Architecture

```
Pre-recorded Video File
        ↓
OpenCV reads frame-by-frame
        ↓
Pre-process frame
(resize 640×640, normalize, transpose)
        ↓
ONNX Runtime — YOLOv8 Nano Inference
(returns bounding boxes, classes, confidence scores)
        ↓                          ↓
Post-process detections      Check line crossing
(filter by confidence,             ↓
 apply NMS)              If vehicle crossed line:
        ↓                  → update in-memory counter
Draw bounding boxes        → push to detection queue
and counting line                  ↓
on frame                  DB Writer Thread (every 1 sec)
        ↓                  bulk_create() → SQLite DB
Encode frame as
JPEG base64
        ↓
Package JSON payload:
{ frame, counts, total, recent_detections }
        ↓
InMemoryChannelLayer
(internal message bus)
        ↓
Django Channels WebSocket Consumer
        ↓
        ↓——— WebSocket (persistent connection) ———↓
                                            Browser
                                    HTML5 Canvas renders frame
                                    Count cards update live
                                    Detection log updates
```

---

## 6. Key Design Decisions

### Why ONNX Runtime Instead of PyTorch?

The AI model was trained using YOLOv8 and saved as a `.pt` (PyTorch) file. Running it directly requires PyTorch — a ~~900MB download. Since contributors in Timor-Leste have slow internet (~~100 KBps), the model was converted once to ONNX format (`best.onnx`). ONNX Runtime is only ~15MB, runs faster than PyTorch on CPU, and requires no deep learning framework.

### Why WebSocket Instead of HTTP?

HTTP requires the browser to repeatedly ask the server for new frames (polling), which causes lag and wastes bandwidth. WebSocket opens one persistent connection and lets the server push frames directly to the browser the moment they are ready — resulting in a smooth live feed.

### Why an In-Memory Counter?

Querying the database for vehicle counts on every frame (30 times per second) would slow the system significantly. Instead, counts are maintained in a Python dictionary in memory and updated instantly. The database is only used for persistence, not for live count reads.

### Why a DB Writer Thread with Bulk Inserts?

Running `Detection.objects.create()` for every detection on every frame would result in ~150 individual database calls per second. Instead, detections are collected in a queue and a separate thread saves them all in a single `bulk_create()` call every 1 second — reducing DB calls from 150/sec to 1/sec.

### Why a Virtual Counting Line?

Without a counting line, a car visible in the video for 4 minutes at 30fps would be counted 7,200 times. The counting line ensures each vehicle is counted exactly once — when its center point crosses the line — regardless of how long it stays in frame. This is how real-world traffic counters work.

---

## 7. Database Design

### Table 1: `VideoSession`

Tracks each video processing run.


| Column      | Type          | Description                |
| ----------- | ------------- | -------------------------- |
| id          | Integer (PK)  | Auto primary key           |
| video_file  | FileField     | Path to uploaded video     |
| started_at  | DateTimeField | When processing started    |
| is_complete | BooleanField  | Whether processing is done |


### Table 2: `Detection`

One row per vehicle that crosses the counting line.


| Column        | Type          | Description                   |
| ------------- | ------------- | ----------------------------- |
| id            | Integer (PK)  | Auto primary key              |
| session       | ForeignKey    | Links to VideoSession         |
| vehicle_class | CharField     | car, truck, motorbike, bus    |
| confidence    | FloatField    | Detection confidence 0.0–1.0  |
| timestamp     | DateTimeField | When the vehicle was detected |


### Relationship

```
VideoSession (1) ————< Detection (many)
```

One session has many detections. Each detection belongs to one session.

---

## 9. Project File Structure

```
# Nanti sei kria 
```

---

## 10. Performance Optimizations


| Optimization                  | Problem it Solves                       | Approach                                |
| ----------------------------- | --------------------------------------- | --------------------------------------- |
| ONNX Runtime                  | PyTorch too heavy (900MB)               | Use ONNX (~15MB), faster on CPU         |
| In-memory counter             | DB queries 30x/sec too slow             | Python dict updated instantly in memory |
| Detection queue + bulk insert | 150 DB writes/sec too slow              | Batch all writes into 1 call per second |
| Virtual counting line         | Same vehicle counted thousands of times | Count only on line crossing             |
| JPEG compression at 70%       | Large frame payload slows WebSocket     | Smaller file size, still visually clear |
| Frame skipping (optional)     | Slow CPU can't keep up at 30fps         | Run inference every 2nd frame           |


---

## 11. Setup & Installation

### Requirements

- Python 3.10+
- `best.onnx` model file (already available)

### Installation

```bash
git clone https://github.com/yourusername/traffic-monitor.git
cd traffic-monitor
python -m venv venv
venv\Scripts\activate          # Windows
source venv/bin/activate        # macOS / Linux
pip install -r requirements.txt
python manage.py migrate
daphne -p 8000 traffic_monitor.asgi:application
```

### requirements.txt

```
django
channels
daphne
opencv-python
numpy
onnxruntime
```

> Total download size for all contributors: ~100MB

---

## 12. Limitations

- Designed for local development only, not production deployment
- Processes pre-recorded video, not live CCTV streams
- Single user — InMemoryChannelLayer does not support multiple server processes
- Counting line position is fixed in code, not configurable from the UI (yet)

---

## 13. Future Scope


| Feature                    | Description                                                                                 |
| -------------------------- | ------------------------------------------------------------------------------------------- |
| Live CCTV stream           | Replace video file input with RTSP stream from a real camera                                |
| Redis channel layer        | Replace InMemoryChannelLayer to support multi-user and deployment                           |
| PostgreSQL                 | Replace SQLite for production-grade database                                                |
| Speed estimation           | Estimate vehicle speed using frame delta and perspective calibration                        |
| Traffic heatmap            | Visualize road congestion from accumulated detection positions                              |
| Edge deployment            | Run ONNX inference on NVIDIA Jetson Nano devices at each camera location for city-scale use |
| Configurable counting line | Let user drag the counting line position on the dashboard                                   |
| Alerts                     | Notify when vehicle count exceeds a set threshold                                           |


