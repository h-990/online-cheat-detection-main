# The-Online-Exam-Proctor

The Online Exam Proctor System is a Flask-based web application for monitoring remote exam sessions using browser webcam capture, computer vision, behavior analysis, and warning/violation tracking.

This repository contains both the exam platform and the live proctoring pipeline:
- student-side exam workflow
- admin-side live monitoring dashboard
- camera frame ingestion and background detection workers
- warning and violation management
- result storage and review

The original project goal remains the same: preserve online exam integrity by detecting suspicious behavior such as missing face, multiple faces, object usage, gaze distraction, movement, tab switching, and voice/noise activity.

## Project Status

This codebase is currently in active repair and stabilization.

Recent work has focused on fixing a major real-time monitoring issue:
- browser frames were arriving, but the admin stream could fall back to raw frames without overlays
- processed frames and raw frames were not clearly separated
- detector status was not continuously visible
- some detector branches were enabled in code but effectively inactive in practice

The README below documents both:
- the intended architecture
- the issues that have been observed in the current implementation

## Original Project Overview

The Online Exam Proctor System is a computer vision-based project designed to ensure the integrity and fairness of online exams. As remote learning and online education grow, the need for a robust proctoring system becomes crucial to prevent cheating and maintain the credibility of the examination process. This project uses computer vision and AI technologies to monitor and analyze students' behavior during the examination and detect suspicious activities. It also includes voice/noise monitoring to strengthen anti-cheating enforcement.

## System Architecture

The project has 2 primary user roles:
- `Student`
- `Admin`

High-level architecture:

1. Student opens the Flask web app and logs in.
2. Student completes system checks and starts the exam.
3. Student browser captures webcam frames using `getUserMedia`.
4. Browser sends frames to Flask `/api/student-frame`.
5. Flask decodes and stores the latest raw frame for that student.
6. A background worker processes the latest frame:
   - face presence and landmarks
   - gaze / eyes closed
   - object detection
   - movement / left seat
   - camera blocked / stale feed
7. Processed state is stored in memory for:
   - admin MJPEG live stream
   - warning generation
   - violation persistence
8. Admin opens the live dashboard and monitors active students.
9. Warnings and final exam outcomes are written to MySQL.

## End-to-End Monitoring Pipeline

### 1. Student Browser Capture

The student exam page:
- opens webcam with `navigator.mediaDevices.getUserMedia`
- renders video frames to a hidden canvas
- exports frames as JPEG base64
- uploads them to `/api/student-frame`
- attempts to maintain a stable frame rate using `requestAnimationFrame`

Current target resolution:
- `640x480`

### 2. Flask Frame Receive Layer

Route:
- `/api/student-frame`

Responsibilities:
- receive image payload from browser
- decode base64 into OpenCV frame
- normalize frame size
- update `latest_student_frames`
- schedule background detection

Important design rule:
- heavy AI inference should not run inside the request handler

### 3. Background Detection Layer

Main worker:
- `_run_student_frame_detection(...)` in `app.py`

Responsibilities:
- object detection
- face / landmark analysis
- eye closure and gaze analysis
- pose and movement analysis
- camera blocked / stale frame checks
- warning trigger decisions
- processed-frame generation for admin stream

### 4. Live Stream Layer

Admin live stream route:
- `/admin/live/<student_id>`

Responsibilities:
- serve MJPEG stream
- prefer processed frame when fresh
- fall back to raw frame when needed
- preserve overlays and detector status even on fallback frames

### 5. Warning / Violation Layer

The system tracks:
- live warnings
- warning cooldowns
- persisted violations
- exam termination thresholds

Examples of mapped events:
- `NO_FACE`
- `MULTIPLE_FACES`
- `DISTRACTION`
- `STUDENT_LEFT_SEAT`
- `VOICE_DETECTED`

### 6. Admin Review Layer

Admin can:
- monitor active students
- inspect warnings
- view results
- terminate an exam
- review recordings / violation history

## Tech Stack

### Backend
- Python
- Flask
- Flask-MySQLdb
- Flask-SocketIO
- MySQL

### Computer Vision / AI
- OpenCV
- MediaPipe
- Ultralytics YOLOv8
- face_recognition / dlib
- pytesseract (optional OCR)

### Frontend
- HTML
- CSS
- JavaScript
- Bootstrap-based UI

### Other Utilities
- PyAutoGUI
- PyGetWindow

## Main Features

### 1. Website Application

Student-side pages:
- Login
- Rules and Regulations
- System Compatibility Check
- Face Input Collection
- Exam
- Result

Admin-side pages:
- Student Listing
- Live Monitoring Dashboard
- Exam Results
- Result Details
- Recordings / Violations review

### 2. Face Detection and Identity Monitoring

- face presence detection
- multiple face detection
- face verification support
- liveness-related checks
- landmark-based eye and gaze analysis

### 3. Movement and Behavior Monitoring

- looking away
- eyes closed for too long
- suspicious movement
- left-seat detection
- camera blocked / covered checks

### 4. Object Detection

Targeted prohibited items include:
- cell phone
- laptop
- tablet
- book / notebook
- headphones / earbuds

### 5. Exam Integrity Controls

- prohibited keys detection
- tab switching / leaving exam interface
- admin warning escalation
- termination after repeated violations

### 6. Audio Monitoring

- voice / noise detection
- audio-based warning generation

## Current Architecture in the Codebase

### Core Application File

Primary backend entrypoint:
- `app.py`

This file currently contains:
- Flask routes
- model initialization
- frame pipeline
- admin live stream
- violation persistence logic
- warning integrations

### Other Important Files

- `admin_live_monitoring.py`
  - additional monitoring utilities
  - audio monitor
  - admin live features

- `warning_system.py`
  - warning counters
  - violation throttling
  - termination thresholds

- `templates/Exam.html`
  - student exam page
  - browser webcam upload logic

- `templates/admin_live_dashboard.html`
  - admin live monitoring UI

- `utils.py`
  - legacy / alternate CV utilities
  - helper detection routines

## Data Flow in Detail

### Student to Backend

`Browser camera -> Canvas -> JPEG base64 -> /api/student-frame`

### Backend Runtime State

In-memory structures include:
- `latest_student_frames`
  - latest raw frame
  - latest processed frame
  - timestamps
  - object labels
  - status snapshot

- `student_detection_state`
  - worker state
  - pending frame
  - violation timers
  - pose history
  - object history

### Backend to Admin

`latest_student_frames -> /admin/live/<student_id> -> MJPEG img stream`

### Backend to Database

Persistent storage includes:
- students
- exam sessions
- results
- violations

## Known Issues We Have Been Facing

This section documents the main problems observed in the current system before recent fixes.

### Issue 1. Processed overlays disappeared after a few seconds

Symptom:
- the admin stream initially showed face/object/diagnostic lines
- after several seconds, the lines disappeared

Root cause:
- the admin stream sometimes served raw frames when processed frames lagged
- raw frames originally had no overlay/status rendering

Impact:
- it looked like detection stopped even when the worker was still active

### Issue 2. Detection pipeline looked frozen

Symptom:
- movement or new objects were not reflected immediately
- admin feed could look stale

Root cause:
- raw frame updates and processed frame updates were not clearly separated
- latest-frame replacement behavior was not strong enough in earlier flow

### Issue 3. Detector visibility was poor

Symptom:
- lines appeared only when warning thresholds were active
- when no warning was active, the feed looked almost blank

Root cause:
- overlay logic mixed detector status with warning state

Impact:
- difficult to debug whether detectors were alive or inactive

### Issue 4. Movement analysis was effectively inactive in practice

Symptom:
- movement / left-seat lines did not appear reliably

Root cause:
- pose analysis depended on runtime flags and was not clearly exposed in UI

### Issue 5. Face presence and landmark quality were conflated

Symptom:
- when landmarks failed, the system often behaved as if no face existed

Root cause:
- face presence and landmark success were too tightly coupled

Impact:
- partial occlusion and low light produced misleading `NO_FACE` behavior

### Issue 6. Low-light performance is still weak

Symptom:
- noisy or dark webcam feed reduces all detector quality

Root cause:
- MediaPipe, YOLO, and fallback detectors all degrade significantly in poor lighting

Impact:
- reduced reliability for objects, gaze, and face landmarks

## Recent Fixes Applied

The following improvements have been made in the current branch:

### Frame Pipeline
- separated raw frame storage from processed frame storage
- normalized frame size to `640x480`
- improved latest-frame scheduling behavior

### Admin Stream
- preserved overlays even when raw-frame fallback is used
- added status snapshot overlay support on fallback frames

### Detector Visibility
- added continuous on-frame status text
- added face count, object labels, person count, prohibited labels
- added pose ON/OFF visibility

### Face Pipeline
- separated `face_detected` from `landmarks_detected`
- added `face_obscured` state
- reduced false `NO_FACE` escalation during obscuration

### Diagnostics
- added `/api/object-detection-status`
- added `/api/detection-pipeline-status`

## Debugging and Validation Guide

### 1. Check model status

Admin route:
- `/api/object-detection-status`

Use it to verify:
- YOLO loaded or not
- model path
- device
- prohibited class mapping

### 2. Check live pipeline status

Admin route:
- `/api/detection-pipeline-status`

Use it to verify:
- frames received
- raw frame age
- processed frame age
- worker running state
- object labels
- person count
- status snapshot

### 3. Check the admin MJPEG stream

Admin route:
- `/admin/live/<student_id>`

Expected continuous overlay:
- face state
- landmark state
- occlusion state
- gaze / eyes state
- object labels
- person count
- pose state

### 4. Check console logs

Important log families:
- frame received
- object inference
- face detection
- no face reset/fire
- camera blocked
- warning generation

## Backend Processing Summary

### When a student frame arrives

1. Browser uploads frame.
2. Flask decodes frame.
3. Raw frame is stored in memory.
4. Detection worker is scheduled.
5. Worker analyzes frame.
6. Processed frame and status snapshot are stored.
7. Admin stream renders processed frame or raw fallback with snapshot overlay.

### When a warning is triggered

1. Detector threshold is exceeded.
2. Warning system checks cooldown.
3. Warning is added to in-memory state.
4. Violation may be persisted.
5. Exam may terminate at threshold.

## Setup and Running

### Requirements

Install dependencies from:
- `requirements.txt`

### Basic Run

1. Clone the repository.
2. Create and activate a Python virtual environment.
3. Install dependencies.
4. Configure MySQL database.
5. Run:

```bash
python app.py
```

### Recommended Environment Notes

- webcam access must be allowed in browser
- MySQL must be running
- YOLO model file should be present
- better lighting improves detection quality significantly

## High-Level Repository Structure

```text
.
|-- app.py
|-- admin_live_monitoring.py
|-- warning_system.py
|-- utils.py
|-- templates/
|-- static/
|-- models/
|-- Haarcascades/
|-- requirements.txt
```

## Future Improvement Areas

- split `app.py` into modules
- separate detection engine from Flask routes
- dedicated face detector independent from FaceMesh
- stronger occlusion detection
- stronger hidden-object detection
- better admin health dashboard
- test suite for frame pipeline and warning logic

## Notes

- This repository still contains legacy and current monitoring logic side by side.
- Some behavior may differ between older helper modules and the active `app.py` runtime path.
- Low-light scenes remain a practical limitation even after pipeline fixes.

## Reference

Project details PDF:
- `OEP Project.pdf`
