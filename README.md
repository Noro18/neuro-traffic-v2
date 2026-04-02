# Neuro-Traffic V2

Projetu Final ba Software Engineering 

# Software Description

**Neuro Traffic** nu'udar real time wbe applicaiton monitoring system ne'ebe kria ho DJango, YOLOv8 Nano, no WebSockets. ne'ebe processa video footage pre-recorded (development) no sei detekta no klasifika veiculu sira uza model YOLOv8 ne'eb train ona atu display video anotady iha live dashbaord, no **logs** deteksaun sira ba iha databse ho live count ba veiculu sira \

# 📸 Features
- Live Anotation ho bounding boxes husi YOLOv8
- Deteksaun no klasifikasaun veiculus (SUV, motor, kareta, tricycle, bemo, taxi, truck).
- Logs ba kada deteksaaun sei rai nu'udar dados iha database
- Live vehivle count iha dashbaord
- Powered husi web socketes

# Architecture **Overview**

![Architecture](docs/images/Traffic%20Monitoring%20System%20Architecture.png)

For further informatino refer to [Architecture](docs/ARCHITECTURE.md)
For Project Overview check out [ProjectPlan](docs/PROJECT_PLAN.md)

# 🛠️ Tech Stack 

| Layer | Technology |
|---|---|
| Web Framework | Django 4.x |
| Real-time Communication | Django Channels (WebSocket) |
| Channel Layer | InMemoryChannelLayer |
| ML Inference | YOLOv8 Nano (Ultralytics) |
| Video Processing | OpenCV (`cv2`) |
| Database | SQLite (Django ORM) |
| Frontend Styling | Tailwind CSS |
| Frontend Rendering | HTML5 Canvas + Vanilla JavaScript |

# 🫂 Kontribuisaun



Refere ba file ida ne'e [Oinsa atu halo kontribuisaun ba Projetu ida ne'e](docs/CONTRIBUTION.md)