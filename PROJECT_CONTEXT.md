# Traffic Monitoring System — Full Project Context

## Project Title
**Real-Time Vehicle Detection and Traffic Monitoring System Using YOLOv8**

---

## Project Overview

A real-time traffic monitoring web application built as a 4-month Software Engineering course project. The system processes pre-recorded video footage, detects and classifies vehicles using a custom-trained YOLOv8 Nano model (`.pt` format), displays the annotated video feed live on a web dashboard, and logs all vehicle detections to a database with live vehicle counts.

---

## How the App Works (End to End)

1. User opens the web dashboard in a browser
2. User uploads or selects a pre-recorded traffic video
3. User clicks **Start Processing**
4. Django starts a background Python thread
5. The background thread reads the video frame by frame using OpenCV
6. Each frame is passed through the Ultralytics YOLOv8 inference engine
7. Ultralytics returns bounding boxes, class labels, and confidence scores for each detected vehicle
8. Bounding boxes are drawn onto the frame manually using OpenCV
9. The vehicle's center point is checked against a virtual counting line
10. If a vehicle crosses the counting line it is added to an in-memory counter and pushed to a detection queue
11. A separate DB writer thread flushes the detection queue to SQLite every 1 second using bulk_create()
12. The annotated frame is encoded as a base64 JPEG string
13. The frame and live counts are sent through InMemoryChannelLayer to the Django Channels WebSocket consumer
14. The WebSocket consumer pushes the payload to the browser
15. JavaScript on the browser receives the payload, draws the frame on an HTML5 Canvas element, and updates the vehicle count cards live

---

## Architecture Overview

```
Pre-recorded Video File
        ↓
OpenCV reads frame-by-frame
        ↓
Ultralytics YOLOv8 Nano inference on each frame
(YOLOv8 Nano .pt model)
        ↓                         ↓
Draw bounding boxes          Check line crossing
on frame using OpenCV        ↓
        ↓              Update in-memory counter
Encode frame as              ↓
JPEG base64             Push to detection queue
        ↓                         ↓
        ↓              DB writer thread (every 1 sec)
        ↓              bulk_create() → SQLite DB
        ↓
InMemoryChannelLayer
(internal message bus between thread and consumer)
        ↓
Django Channels WebSocket Consumer
(server side of WebSocket)
        ↓
        ↓ WebSocket connection (persistent, browser ↔ server)
        ↓
Browser — HTML5 Canvas
        ↓                    ↓
Renders annotated     Live vehicle count
video frame           cards update
```

---

## Tech Stack

| Layer                    | Technology                        |
|--------------------------|-----------------------------------|
| Web Framework            | Django 4.x                        |
| Real-time Communication  | Django Channels (WebSocket)       |
| ASGI Server              | Daphne                            |
| Channel Layer            | InMemoryChannelLayer              |
| ML Inference Engine      | Ultralytics (YOLOv8)              |
| Model Format             | PyTorch (.pt)                     |
| Video Processing         | OpenCV (cv2)                      |
| Frame Data Handling      | NumPy                             |
| Database                 | SQLite (via Django ORM)           |
| Frontend Styling         | Tailwind CSS                      |
| Frontend Rendering       | HTML5 Canvas + Vanilla JavaScript |

---

## Why Ultralytics Directly Instead of ONNX

The trained model is a YOLOv8 Nano `.pt` file (PyTorch format). The project runs inference directly using the Ultralytics library, which handles preprocessing, NMS, and postprocessing automatically — reducing boilerplate code significantly.

```python
from ultralytics import YOLO

model = YOLO("best.pt")
results = model(frame)
```

This requires PyTorch and Ultralytics but is simpler to maintain and gives access to the full Ultralytics API including built-in NMS, confidence filtering, and class label mapping. There is no need for manual preprocessing or postprocessing steps.

**Note:** An earlier version of this project planned to use ONNX Runtime to avoid the large PyTorch download (~900MB) for contributors with slow internet. Since connectivity is no longer a constraint, the project now uses Ultralytics directly with the `.pt` model file.

---

## Component Breakdown

### 1. OpenCV (`cv2`)
**Role:** Reads the pre-recorded video file frame by frame.

OpenCV opens the `.mp4` file and on each loop iteration returns one raw frame as a NumPy array (a 3D array of pixel values with shape [height, width, 3] for BGR channels). It also draws bounding boxes and the counting line on frames.

Key operations:
- `cv2.VideoCapture(path)` — opens the video file
- `cap.read()` — reads the next frame, returns (success_bool, frame)
- `cv2.imencode('.jpg', frame)` — encodes frame as JPEG bytes
- `cv2.line()` — draws the virtual counting line on the frame
- `cv2.rectangle()` — draws bounding boxes on the frame
- `cv2.putText()` — draws class labels on the frame

---

### 2. Ultralytics YOLOv8
**Role:** Runs inference on each frame using the YOLOv8 Nano `.pt` model.

Ultralytics loads the `best.pt` file and runs the neural network forward pass on each frame to produce detections. It automatically handles preprocessing, NMS, and postprocessing — returning ready-to-use bounding boxes, confidence scores, and class IDs.

Key operations:
- `YOLO("models/best.pt")` — loads the model
- `model(frame)` — runs inference and returns results
- `results[0].boxes` — access bounding boxes, confidence scores, and class IDs

```python
from ultralytics import YOLO

model = YOLO("models/best.pt")

results = model(frame)
for box in results[0].boxes:
    x1, y1, x2, y2 = map(int, box.xyxy[0])
    confidence = float(box.conf[0])
    class_id = int(box.cls[0])
    label = model.names[class_id]
```

No manual preprocessing or NMS required — Ultralytics handles it all internally.

---

### 3. Python Background Thread
**Role:** Runs the entire video processing loop without blocking Django.

Django's main process handles HTTP requests and WebSocket connections. If video processing ran inside Django directly it would block everything. A separate daemon thread runs the OpenCV + Ultralytics loop independently.

```python
import threading

thread = threading.Thread(target=process_video, args=(video_path, session_id), daemon=True)
thread.start()
```

The thread is a daemon thread — it automatically stops when the main program exits.

---

### 4. Virtual Counting Line
**Role:** Ensures each vehicle is counted exactly once, not once per frame.

A horizontal line is drawn across the video at a fixed Y coordinate. A vehicle is counted only when its bounding box center point crosses from above the line to below it (or vice versa) between consecutive frames.

Without this, a car visible for 4 minutes at 30fps would be counted 7,200 times. With line crossing, it is counted exactly once.

```python
LINE_Y = 300  # y pixel coordinate of the counting line

def check_line_crossing(prev_center_y, curr_center_y):
    if prev_center_y < LINE_Y and curr_center_y >= LINE_Y:
        return True  # vehicle crossed the line downward
    return False
```

Previous center positions are stored in a dictionary keyed by approximate horizontal position to loosely associate detections across frames.

---

### 5. In-Memory Vehicle Counter
**Role:** Keeps live vehicle counts without querying the database on every frame.

A thread-safe counter using a Python dictionary and a threading lock. The inference thread updates this counter instantly without any database involvement. The browser receives counts from this in-memory dictionary, not from a DB query, making it extremely fast.

```python
from collections import defaultdict
import threading

class VehicleCounter:
    def __init__(self):
        self.counts = defaultdict(int)
        self.lock = threading.Lock()

    def add(self, vehicle_class):
        with self.lock:
            self.counts[vehicle_class] += 1

    def get_counts(self):
        with self.lock:
            return dict(self.counts)

    def reset(self):
        with self.lock:
            self.counts.clear()
```

---

### 6. Detection Queue + DB Writer Thread
**Role:** Decouples database writes from inference to avoid slowing down the video loop.

Instead of calling `Detection.objects.create()` for every detection (which would be ~150 individual DB calls per second at 30fps with 5 vehicles per frame), detections are pushed to a shared `deque` queue. A separate thread wakes up every 1 second, grabs everything accumulated in the queue, and saves it all in one `bulk_create()` call.

```python
from collections import deque
import threading, time

detection_queue = deque()
queue_lock = threading.Lock()

def db_writer_worker():
    while True:
        time.sleep(1)
        with queue_lock:
            if not detection_queue:
                continue
            batch = list(detection_queue)
            detection_queue.clear()
        Detection.objects.bulk_create([
            Detection(
                session_id=d["session_id"],
                vehicle_class=d["class"],
                confidence=d["confidence"]
            ) for d in batch
        ])
```

| Approach | DB calls/sec | Performance |
|---|---|---|
| `create()` per detection | 150 | Slow |
| `bulk_create()` per frame | 30 | Fast |
| Queue + `bulk_create()` every 1 sec | 1 | Fastest |

---

### 7. InMemoryChannelLayer
**Role:** Internal message bus between the background thread and the WebSocket consumer.

The video processing thread and the Django Channels consumer are two separate execution contexts. They cannot call each other directly. InMemoryChannelLayer is a shared in-memory queue that sits between them. The thread puts messages in, the consumer takes messages out and forwards them to the browser.

No Redis or external broker is needed. InMemoryChannelLayer is built into Django Channels and works entirely in process memory. It is suitable for single-user local development only — it does not persist across restarts and does not support multiple server processes.

Configuration in `settings.py`:
```python
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels.layers.InMemoryChannelLayer"
    }
}
```

---

### 8. Django Channels + WebSocket Consumer
**Role:** Manages the persistent WebSocket connection between the server and browser.

Normal Django HTTP closes the connection after each response. WebSocket keeps one connection permanently open so the server can push data to the browser at any time without the browser asking.

The consumer is a Python class with three lifecycle methods:
- `connect()` — called when browser opens the WebSocket
- `disconnect()` — called when browser closes the WebSocket
- `receive()` — called when browser sends a message to the server
- `send_frame()` — custom method called by the channel layer to forward frames

```python
from channels.generic.websocket import AsyncWebsocketConsumer
import json

class VideoConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.group_name = "video_stream"
        await self.channel_layer.group_add(self.group_name, self.channel_name)
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(self.group_name, self.channel_name)

    async def send_frame(self, event):
        await self.send(text_data=json.dumps(event["data"]))
```

---

### 9. WebSocket Payload
**Role:** The data structure sent from the server to the browser on every frame.

```json
{
  "frame": "<base64 encoded JPEG string>",
  "counts": {
    "car": 18,
    "truck": 4,
    "motorbike": 12,
    "bus": 2
  },
  "total": 36,
  "recent_detections": [
    { "class": "car",   "confidence": 0.94, "time": "12:00:05" },
    { "class": "truck", "confidence": 0.87, "time": "12:00:08" }
  ]
}
```

---

### 10. HTML5 Canvas + JavaScript
**Role:** Renders each incoming frame on screen and updates count cards.

The browser opens a WebSocket connection on page load. Every time a message arrives, JavaScript decodes the base64 JPEG into an Image object and draws it onto a `<canvas>` element. This is faster than swapping `<img>` src tags because canvas rendering is direct and non-blocking.

```javascript
const canvas = document.getElementById('videoCanvas');
const ctx = canvas.getContext('2d');
const socket = new WebSocket('ws://localhost:8000/ws/stream/');

socket.onmessage = (event) => {
    const data = JSON.parse(event.data);

    // draw frame
    const img = new Image();
    img.onload = () => ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
    img.src = 'data:image/jpeg;base64,' + data.frame;

    // update count cards
    document.getElementById('car-count').textContent    = data.counts.car    || 0;
    document.getElementById('truck-count').textContent  = data.counts.truck  || 0;
    document.getElementById('bike-count').textContent   = data.counts.motorbike || 0;
    document.getElementById('bus-count').textContent    = data.counts.bus    || 0;
    document.getElementById('total-count').textContent  = data.total         || 0;
};
```

---

### 11. SQLite Database
**Role:** Persists all detection events for querying, history, and reporting.

SQLite is Django's default database. It is a file-based database requiring zero configuration, stored as a single `.db` file in the project root. It is suitable for a local course project. At city scale this would be replaced with PostgreSQL.

---

### 12. Daphne
**Role:** The ASGI server that replaces Django's default `runserver` to support WebSockets.

Django's default development server only supports WSGI (HTTP only). WebSockets require ASGI. Daphne is the official ASGI server for Django Channels. It handles both HTTP and WebSocket connections simultaneously.

Run the app with:
```bash
daphne -p 8000 traffic_monitor.asgi:application
```

Instead of the usual:
```bash
python manage.py runserver  # this does NOT support WebSockets
```

---

## Database Schema

### Table 1: `VideoSession`
Tracks each video processing run.

| Column | Type | Description |
|---|---|---|
| `id` | Integer (PK) | Auto-incremented primary key |
| `video_file` | FileField | Path to uploaded video file |
| `started_at` | DateTimeField | When processing started |
| `is_complete` | BooleanField | Whether processing has finished |

### Table 2: `Detection`
One row per vehicle that crosses the counting line.

| Column | Type | Description |
|---|---|---|
| `id` | Integer (PK) | Auto-incremented primary key |
| `session` | ForeignKey → VideoSession | Which session this belongs to |
| `vehicle_class` | CharField | car, truck, motorbike, bus |
| `confidence` | FloatField | YOLO confidence score 0.0–1.0 |
| `timestamp` | DateTimeField | When the vehicle was detected |

### Relationship
One `VideoSession` has many `Detection` rows. Each `Detection` belongs to exactly one `VideoSession`.

### Django Models Code
```python
from django.db import models

class VideoSession(models.Model):
    video_file  = models.FileField(upload_to='videos/')
    started_at  = models.DateTimeField(auto_now_add=True)
    is_complete = models.BooleanField(default=False)

    def __str__(self):
        return f"Session {self.id} - {self.video_file}"


class Detection(models.Model):
    VEHICLE_CHOICES = [
        ('car',       'Car'),
        ('truck',     'Truck'),
        ('motorbike', 'Motorbike'),
        ('bus',       'Bus'),
    ]

    session       = models.ForeignKey(VideoSession, on_delete=models.CASCADE)
    vehicle_class = models.CharField(max_length=50, choices=VEHICLE_CHOICES)
    confidence    = models.FloatField()
    timestamp     = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"{self.vehicle_class} ({self.confidence:.2f}) @ {self.timestamp}"
```

### Key Queries
```python
# live counts for current session
from django.db.models import Count

counts = Detection.objects.filter(
    session_id=current_session_id
).values('vehicle_class').annotate(total=Count('id'))

# total vehicles for current session
total = Detection.objects.filter(session_id=current_session_id).count()

# last 10 detections for detection log
recent = Detection.objects.filter(
    session_id=current_session_id
).order_by('-timestamp')[:10]
```

---

## Project File Structure

```
traffic_monitor/
├── manage.py
├── requirements.txt
├── db.sqlite3                        # auto-created after migrate
├── models/
│   └── best.pt                       # YOLOv8 Nano PyTorch model
├── media/
│   └── videos/                       # uploaded video files stored here
├── core/
│   ├── migrations/
│   ├── __init__.py
│   ├── models.py                     # VideoSession, Detection models
│   ├── consumers.py                  # WebSocket consumer
│   ├── video_processor.py            # OpenCV + Ultralytics background thread
│   ├── views.py                      # dashboard view, video upload
│   ├── urls.py
│   └── templates/
│       └── core/
│           └── dashboard.html        # main UI with canvas + tailwind
└── traffic_monitor/
    ├── __init__.py
    ├── settings.py                   # Django settings + Channels config
    ├── urls.py
    ├── asgi.py                       # ASGI + Channels routing config
    └── routing.py                    # WebSocket URL routing
```

---

## Environment Setup

### Prerequisites
- Python 3.10+
- pip
- `best.pt` — your trained YOLOv8 Nano model file

### Installation for All Contributors
```bash
git clone https://github.com/yourusername/traffic-monitor.git
cd traffic-monitor
python -m venv venv

# Windows
venv\Scripts\activate
# macOS / Linux
source venv/bin/activate

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
ultralytics
```

---

## Key Configuration Files

### `settings.py` additions
```python
INSTALLED_APPS = [
    ...
    'channels',
    'core',
]

# Use Channels as the ASGI application
ASGI_APPLICATION = 'traffic_monitor.asgi.application'

# InMemoryChannelLayer - no Redis needed for local dev
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels.layers.InMemoryChannelLayer"
    }
}

# Media files for uploaded videos
import os
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

### `asgi.py`
```python
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
import core.routing

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'traffic_monitor.settings')

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": AuthMiddlewareStack(
        URLRouter(core.routing.websocket_urlpatterns)
    ),
})
```

### `routing.py`
```python
from django.urls import re_path
from core import consumers

websocket_urlpatterns = [
    re_path(r'ws/stream/$', consumers.VideoConsumer.as_asgi()),
]
```

---

## Dashboard UI Layout

```
┌─────────────────────────────────────────────────────┐
│  🚦 Traffic Monitoring System                        │
├─────────────────────────────────────────────────────┤
│                                                      │
│   🎥 Live Video Feed (HTML5 Canvas)                  │
│   [ annotated frames with bounding boxes ]          │
│   [ green counting line drawn across road ]         │
│                                                      │
├──────────────┬──────────────┬──────────┬────────────┤
│  🚗 Cars     │  🚛 Trucks   │ 🏍 Bikes │ 🚌 Buses  │
│      18      │      4       │    12    │     2      │
├─────────────────────────────────────────────────────┤
│  Total Vehicles: 36                                  │
├─────────────────────────────────────────────────────┤
│  📋 Recent Detections                                │
│  car · 0.94 · 12:00:05                              │
│  truck · 0.87 · 12:00:08                            │
│  motorbike · 0.91 · 12:00:12                        │
└─────────────────────────────────────────────────────┘
```

---

## Performance Optimizations

### 1. Ultralytics Built-in Pipeline
Ultralytics handles preprocessing, NMS, and postprocessing internally — no manual implementation needed. This reduces code complexity and potential bugs compared to a manual ONNX pipeline.

### 2. Bulk Database Writes
All detections within a 1-second window are saved in a single `bulk_create()` call instead of individual `create()` calls. Reduces DB calls from ~150/sec to 1/sec.

### 3. In-Memory Vehicle Counter
Live counts shown on the dashboard come from a Python dictionary in memory, not from a database query. This makes count updates instant and zero-cost.

### 4. Frame Skipping (optional)
If inference is too slow on low-end hardware, only run inference every 2nd or 3rd frame:
```python
if frame_count % 2 != 0:
    continue  # skip inference on this frame
```

### 5. JPEG Compression Quality
Lower JPEG quality reduces base64 payload size and speeds up WebSocket transfer:
```python
encode_params = [cv2.IMWRITE_JPEG_QUALITY, 70]  # 70% quality, smaller size
_, buffer = cv2.imencode('.jpg', frame, encode_params)
```

---

## Vehicle Counting Logic (Line Crossing)

A vehicle is counted exactly once when its bounding box center point crosses the virtual counting line. This prevents a car visible for 4 minutes from being counted thousands of times.

```
Video frame:

  🚗 (center_y = 250, above line)   → not counted yet
  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━    ← LINE_Y = 300
  🚗 (center_y = 320, below line)   → COUNTED ✅ (crossed from above to below)
```

The line is drawn visually on every frame so users can see it.

---

## Glossary

| Term | Meaning |
|---|---|
| **YOLOv8 Nano** | Smallest and fastest variant of YOLOv8, good for CPU inference |
| **Ultralytics** | Python library that provides the YOLOv8 API including inference, NMS, and postprocessing |
| **.pt file** | PyTorch model weights file — loaded directly by Ultralytics |
| **WebSocket** | Protocol for persistent two-way connection between browser and server |
| **Django Channels** | Django extension that adds WebSocket support |
| **InMemoryChannelLayer** | In-process message bus for passing data between threads and consumers |
| **Daphne** | ASGI server required to run Django Channels (replaces runserver) |
| **ASGI** | Async Server Gateway Interface — Django's async protocol layer |
| **Canvas** | HTML element for drawing graphics with JavaScript |
| **base64** | Encoding scheme that converts binary data (JPEG) to a text string for JSON transfer |
| **bulk_create()** | Django ORM method to insert many rows in a single SQL statement |
| **NMS** | Non-Maximum Suppression — removes duplicate bounding boxes, handled automatically by Ultralytics |
| **Bounding Box** | Rectangle coordinates [x1, y1, x2, y2] around a detected vehicle |
| **Confidence Score** | How certain YOLO is about a detection, 0.0 to 1.0 |
| **Line Crossing** | Technique to count a vehicle exactly once when it crosses a virtual line |
| **Edge Computing** | Running AI inference at the camera location instead of a central server |
| **daemon=True** | Makes a thread automatically stop when the main program exits |

---

## What This Project Is NOT

- Not a production system
- Not designed for live CCTV streams (uses pre-recorded video)
- Not multi-user (InMemoryChannelLayer is single-process only)
- Not deployable to the cloud without replacing InMemoryChannelLayer with Redis
- Not city-scale (city-scale requires edge computing devices at each camera, distributed infrastructure, and fiber/4G networking)

---

## Future Scope (Beyond Course Project)

- Replace pre-recorded video with live RTSP CCTV stream
- Replace InMemoryChannelLayer with Redis for multi-user support
- Replace SQLite with PostgreSQL for production
- Deploy edge devices (NVIDIA Jetson Nano ~$150 each) at each camera
- Each edge device runs its own inference locally
- Only sends count data (not video) to the central server — saves 99% bandwidth
- Add speed estimation using frame delta and perspective calibration
- Add traffic density heatmap from accumulated bounding box positions
- Retrain YOLO model on Timorese traffic data for better local accuracy