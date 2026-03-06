<![CDATA[<div align="center">

# 🛡️ AI Vision — Online Exam Proctor System

**An AI-powered real-time online exam proctoring platform that uses computer vision, deep learning, and behavioral analysis to ensure exam integrity.**

[![Python](https://img.shields.io/badge/Python-3.9+-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://python.org)
[![Flask](https://img.shields.io/badge/Flask-2.3-000000?style=for-the-badge&logo=flask&logoColor=white)](https://flask.palletsprojects.com)
[![OpenCV](https://img.shields.io/badge/OpenCV-4.8-5C3EE8?style=for-the-badge&logo=opencv&logoColor=white)](https://opencv.org)
[![MediaPipe](https://img.shields.io/badge/MediaPipe-0.10-00A67E?style=for-the-badge&logo=google&logoColor=white)](https://mediapipe.dev)
[![YOLOv8](https://img.shields.io/badge/YOLOv8-Ultralytics-FF6F00?style=for-the-badge&logo=yolo&logoColor=white)](https://ultralytics.com)
[![MySQL](https://img.shields.io/badge/MySQL-MariaDB-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](https://mysql.com)
[![Socket.IO](https://img.shields.io/badge/Socket.IO-Real--time-010101?style=for-the-badge&logo=socket.io&logoColor=white)](https://socket.io)

---

*Built for educational institutions to conduct secure, remotely proctored examinations with live monitoring, AI-driven violation detection, and comprehensive admin controls.*

</div>

---

## 📑 Table of Contents

- [Overview](#-overview)
- [Key Features](#-key-features)
- [System Architecture](#-system-architecture)
- [Technology Stack](#-technology-stack)
- [Project Structure](#-project-structure)
- [Database Schema](#-database-schema)
- [Installation & Setup](#-installation--setup)
- [Configuration](#-configuration)
- [Student Exam Flow](#-student-exam-flow)
- [Admin Dashboard](#-admin-dashboard)
- [AI Detection Pipeline](#-ai-detection-pipeline)
- [Warning & Violation System](#-warning--violation-system)
- [Real-Time Communication](#-real-time-communication)
- [API Endpoints](#-api-endpoints)
- [Security Features](#-security-features)
- [Troubleshooting](#-troubleshooting)

---

## 🔭 Overview

The **AI Vision Online Exam Proctor** is a full-stack web application that monitors students during online examinations using their webcam feed. The system uses multiple AI models running simultaneously to detect suspicious behavior in real-time.

### What It Does

| Capability | Description |
|---|---|
| **Face Detection** | Ensures the student's face is continuously visible via MediaPipe FaceMesh |
| **Object Detection** | Identifies prohibited items (phones, books, laptops) using YOLOv8 |
| **Gaze Tracking** | Monitors eye position and iris offset to detect looking away |
| **Head Pose Estimation** | Calculates yaw angle to detect head turning |
| **Multiple Person Detection** | Flags when more than one person appears in the frame |
| **Tab Switch Detection** | Detects when students leave the exam browser tab |
| **Audio Monitoring** | Captures microphone input to detect voice/noise activity |
| **Live Admin Monitoring** | Streams student webcam feeds to an admin dashboard in real-time |

### How It Works (High Level)

```
Student Browser                        Flask Server                          Admin Dashboard
┌─────────────────┐                 ┌─────────────────────┐                ┌──────────────────┐
│  Webcam Capture  │  Base64 JPEG   │  Frame Decode       │  Socket.IO     │  Live Feed       │
│  via getUserMedia├───────────────→│  AI Detection        ├───────────────→│  Warning Toasts  │
│  + Mic Monitor   │  /api/student  │  Warning Engine      │  Events        │  Student Cards   │
│  + Tab Detection │    -frame      │  Violation Persist   │                │  Manual Controls │
└─────────────────┘                 └─────────────────────┘                └──────────────────┘
```

---

## ✨ Key Features

### 🎓 For Students
- **Pre-Exam System Check** — verifies camera, microphone, and browser compatibility
- **Face Registration** — webcam or file upload for identity verification
- **Face Confirmation** — verifies the logged-in student matches the registered face
- **Timed MCQ Exam** — randomized questions with auto-submit on timeout
- **Real-Time Warning Display** — visual alerts shown when violations are detected
- **Termination Screen** — clear feedback when the exam is terminated with reason and timestamp
- **Auto-Save & Submit** — exam progress is saved and submitted automatically

### 🛡️ For Administrators
- **Live Monitoring Dashboard** — view all active students in a grid layout with status indicators
- **Real-Time Toast Notifications** — instant alerts when any student triggers a violation
- **Per-Student Controls** — buttons to **Clear Warnings** or **Force Terminate** individual exams
- **Auto-Terminate Toggle** — master switch to enable/disable automatic exam termination (defaults OFF)
- **Live MJPEG Video Feed** — click to view any student's processed webcam stream with AI overlays
- **Exam Results & History** — detailed per-student result pages with violation breakdown
- **Recordings Review** — access to saved violation recordings and evidence
- **Student Management** — add, register faces, and manage student accounts

---

## 🏗 System Architecture

### End-to-End Data Flow

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          STUDENT BROWSER                                    │
│                                                                              │
│   navigator.mediaDevices.getUserMedia()                                      │
│         │                                                                    │
│         ▼                                                                    │
│   Hidden <video> element → <canvas> → .toDataURL('image/jpeg')              │
│         │                                                                    │
│         ▼                                                                    │
│   requestAnimationFrame loop → POST /api/student-frame (base64 payload)     │
│                                                                              │
│   Tab Switch Detection (visibilitychange) ─→ Socket.IO 'tab_switch_detected'│
│   Microphone Monitor (Web Audio API)       ─→ Socket.IO 'voice_detected'    │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          FLASK SERVER (app.py)                               │
│                                                                              │
│   /api/student-frame                                                         │
│       │                                                                      │
│       ├──→ Decode base64 → OpenCV frame (BGR)                               │
│       ├──→ Store raw frame in latest_student_frames[student_id]             │
│       └──→ Schedule background detection worker                              │
│                                                                              │
│   Background Worker: _run_student_frame_detection()                         │
│       │                                                                      │
│       ├──→ FaceAnalyzer.process_frame()      [MediaPipe FaceMesh]           │
│       │       Returns: face_detected, yaw_angle, EAR, iris_offset           │
│       │                                                                      │
│       ├──→ PersonDetector.process_frame()    [YOLOv8]                       │
│       │       Returns: person_count, bounding_boxes, banned_objects          │
│       │                                                                      │
│       ├──→ DecisionEngine.evaluate()         [Rule Engine]                  │
│       │       Returns: active_warnings[], penalty_score                     │
│       │                                                                      │
│       ├──→ UILayer.draw_overlays()           [OpenCV Drawing]               │
│       │       Draws: bounding boxes, status text, alert panels              │
│       │                                                                      │
│       ├──→ _persist_behavior_violation()     [MySQL + Socket.IO]            │
│       │       Saves violation to DB, emits to admin namespace               │
│       │                                                                      │
│       └──→ Store processed frame in latest_student_frames[student_id]       │
│                                                                              │
│   WarningSystem                                                              │
│       ├──→ Tracks per-student warning counts                                │
│       ├──→ Enforces cooldown between warnings                               │
│       ├──→ Emits Socket.IO events to admin                                  │
│       └──→ Auto-terminates at threshold (if toggle is ON)                   │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         ADMIN DASHBOARD (Students.html)                      │
│                                                                              │
│   On Page Load:                                                              │
│       fetch('/api/admin-active-students') → populate student cards           │
│                                                                              │
│   Socket.IO Events:                                                          │
│       'student_violation'       → show toast + update card                   │
│       'student_needs_review'    → highlight student card                     │
│       'student_exam_terminated' → mark student as terminated                │
│                                                                              │
│   Admin Actions:                                                             │
│       Clear Warnings   → emit('admin_clear_warnings')                       │
│       Force Terminate   → emit('admin_force_terminate')                     │
│       Toggle Auto-Term  → emit('admin_toggle_enforcement')                  │
│       View Live Feed    → open /admin/live/<student_id> (MJPEG)             │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Threading Model

The server runs multiple concurrent threads:

| Thread | Purpose |
|---|---|
| **Main Flask Thread** | Handles HTTP requests and Socket.IO events |
| **Detection Workers** | Per-student background threads for AI inference |
| **Frame Staleness Watchdog** | Monitors for students whose camera feed stops |
| **Audio Monitor** | Processes microphone data from student streams |

---

## 🛠 Technology Stack

### Backend
| Technology | Version | Purpose |
|---|---|---|
| Python | 3.9+ | Core runtime |
| Flask | 2.3.3 | Web framework |
| Flask-SocketIO | 5.3.6 | Real-time bidirectional communication |
| PyMySQL | 1.1.0 | MySQL/MariaDB database driver |
| Eventlet | 0.33.3 | Async networking for Socket.IO |
| bcrypt | 4.1.3 | Password hashing |

### Computer Vision & AI
| Technology | Version | Purpose |
|---|---|---|
| OpenCV | 4.8.1 | Image processing, frame manipulation, HUD rendering |
| MediaPipe | 0.10.7 | Face mesh (468 landmarks), iris tracking, pose estimation |
| Ultralytics YOLOv8 | 8.3.0 | Real-time object detection (phones, books, laptops) |
| YOLOv4-tiny | — | Lightweight fallback object detection |
| Haar Cascades | — | Fast frontal face detection (legacy fallback) |
| pytesseract | 0.3.10 | Optional OCR for text detection in frames |

### Frontend
| Technology | Purpose |
|---|---|
| HTML5 / CSS3 / JavaScript | Core UI |
| Bootstrap 5 | Responsive layout and components |
| Socket.IO Client | Real-time event handling |
| Web Audio API | Microphone monitoring |
| Canvas API | Frame capture and manipulation |
| getUserMedia | Webcam access |

---

## 📁 Project Structure

```
online-cheat-detection-main/
│
├── app.py                          # Main Flask application (3900+ lines)
│                                     # All routes, Socket.IO handlers, frame pipeline,
│                                     # detection workers, admin APIs, auth logic
│
├── face_pipeline.py                # MediaPipe FaceMesh wrapper
│                                     # Thread-safe face detection with global lock
│                                     # Returns: face_detected, yaw_angle, EAR, iris_offset
│
├── person_pipeline.py              # YOLOv8 person & object detector
│                                     # Detects: persons (class 0), phones (67),
│                                     # books (73), laptops (77)
│
├── decision_engine.py              # Rule evaluation engine with temporal debouncing
│                                     # Evaluates all detection signals against thresholds
│                                     # Currently: only face detection rule is active
│
├── warning_system.py               # Per-student warning counter & enforcement
│                                     # Tracks warnings, violations, cooldowns
│                                     # Auto-terminate toggle (defaults OFF)
│
├── config_vision.py                # All tunable parameters
│                                     # Thresholds, debounce timers, penalties
│
├── vision_ui.py                    # OpenCV HUD overlay renderer
│                                     # Draws bounding boxes, status panels, alerts
│
├── admin_live_monitoring.py        # Legacy admin monitoring utilities
│                                     # Audio monitor, violation recording
│
├── proctoring_core.py              # Core proctoring logic helpers
├── utils.py                        # Legacy CV utilities and helper functions
├── create_users.py                 # Script to populate DB with test users
├── download_yolo_models.py         # Script to download YOLO model weights
├── main.py                         # Alternate entrypoint (development)
├── rewrite.py                      # Standalone detection test harness
│
├── requirements.txt                # Python dependencies
├── examproctordb.sql               # MySQL database schema + sample data
├── .gitignore                      # Git ignore rules
├── START_EXAM_PROCTOR.bat          # Windows startup script
├── OEP Project.pdf                 # Project documentation PDF
│
├── models/                         # Pre-trained model files
│   ├── yolov4-tiny.cfg            # YOLOv4-tiny architecture
│   ├── yolov4-tiny.weights        # YOLOv4-tiny weights (23MB)
│   ├── MobileNetSSD_deploy.*      # MobileNet SSD model files
│   └── coco.names                 # COCO class labels
│
├── Haarcascades/                   # OpenCV Haar cascade classifiers
│   └── haarcascade_frontalface_default.xml
│
├── templates/                      # Jinja2 HTML templates (22 files)
│   │
│   │  ─── Student Pages ───
│   ├── login.html                  # Login page (students + admin)
│   ├── signup.html                 # Student registration
│   ├── forgot_password.html        # Password reset request
│   ├── reset_password.html         # Password reset form
│   ├── index.html                  # Landing page
│   ├── ExamRules.html              # Exam rules & regulations
│   ├── ExamSystemCheck.html        # Pre-exam system compatibility check
│   ├── ExamSystemCheckError.html   # System check failure page
│   ├── ExamFaceInput.html          # Face registration (webcam/upload)
│   ├── ExamConfirmFaceInput.html   # Face verification before exam
│   ├── Exam.html                   # Main exam page (150KB, full proctoring client)
│   ├── ExamResult.html             # Post-exam score display
│   ├── ExamResultPass.html         # Pass result page
│   ├── ExamResultFail.html         # Fail result page
│   ├── showResultPass.html         # Detailed pass result view
│   │
│   │  ─── Admin Pages ───
│   ├── Students.html               # Admin dashboard (75KB), student management,
│   │                                 # live monitoring, active student grid
│   ├── Results.html                # Exam results listing
│   ├── ResultDetails.html          # Per-student result breakdown
│   ├── ResultDetailsVideo.html     # Violation video playback
│   ├── Recordings.html             # Saved recordings browser
│   └── admin_live_dashboard.html   # Dedicated live monitoring page
│
├── static/
│   ├── css/                        # Stylesheets (14 files)
│   ├── js/                         # Client-side scripts (8 files)
│   │   └── questions.js            # Exam question bank
│   ├── img/                        # UI assets and icons (22 files)
│   ├── Profiles/                   # Student profile photos (gitignored)
│   └── audio_recordings/           # Exam session recordings (gitignored)
│
└── utils/
    └── coco.txt                    # COCO dataset class names
```

---

## 🗄 Database Schema

The system uses a **MySQL/MariaDB** database named `examproctordb` with 5 tables:

### Entity Relationship

```
┌─────────────┐     ┌────────────────┐     ┌──────────────┐
│  students   │────<│ exam_sessions  │────<│ exam_results │
│             │     │                │     │              │
│ ID (PK)     │     │ SessionID (PK) │     │ ResultID (PK)│
│ Name        │     │ StudentID (FK) │     │ StudentID(FK)│
│ Email (UQ)  │     │ StartTime      │     │ SessionID(FK)│
│ Password    │     │ EndTime        │     │ Score        │
│ Profile     │     │ Status         │     │ Status       │
│ Role        │     └────────┬───────┘     └──────────────┘
└──────┬──────┘              │
       │                     │
       │     ┌───────────────┘
       │     │
       ▼     ▼
┌─────────────────┐     ┌─────────────┐
│   violations    │     │  profiles   │
│                 │     │             │
│ ViolationID(PK) │     │ id (PK)     │
│ StudentID (FK)  │     │ student_id  │
│ SessionID (FK)  │     │ image_path  │
│ ViolationType   │     │ image_type  │
│ Details         │     │ face_detect │
│ Timestamp       │     └─────────────┘
└─────────────────┘
```

### Table Details

#### `students`
| Column | Type | Description |
|---|---|---|
| `ID` | INT (PK, Auto) | Unique student identifier |
| `Name` | VARCHAR(100) | Full name |
| `Email` | VARCHAR(100, Unique) | Login email |
| `Password` | VARCHAR(255) | bcrypt/pbkdf2 hashed password |
| `Profile` | VARCHAR(255) | Profile image filename |
| `Role` | ENUM('ADMIN','STUDENT') | Access role |

#### `exam_sessions`
| Column | Type | Description |
|---|---|---|
| `SessionID` | INT (PK, Auto) | Unique session identifier |
| `StudentID` | INT (FK → students) | Which student |
| `StartTime` | DATETIME | When the exam started |
| `EndTime` | DATETIME | When the exam ended |
| `Status` | ENUM('IN_PROGRESS','COMPLETED','TERMINATED') | Session outcome |

#### `exam_results`
| Column | Type | Description |
|---|---|---|
| `ResultID` | INT (PK, Auto) | Unique result identifier |
| `StudentID` | INT (FK → students) | Which student |
| `SessionID` | INT (FK → exam_sessions) | Which session |
| `Score` | DECIMAL(5,2) | Achieved score |
| `TotalQuestions` | INT | Question count |
| `CorrectAnswers` | INT | Correct count |
| `Status` | ENUM('PASS','FAIL','TERMINATED') | Result type |

#### `violations`
| Column | Type | Description |
|---|---|---|
| `ViolationID` | INT (PK, Auto) | Unique violation identifier |
| `StudentID` | INT (FK → students) | Offending student |
| `SessionID` | INT (FK → exam_sessions) | During which session |
| `ViolationType` | VARCHAR(64) | Category: `NO_FACE`, `MULTIPLE_FACES`, `TAB_SWITCH`, `VOICE_DETECTED`, `PROHIBITED_OBJECT`, `DISTRACTION`, `CAMERA_OFF` |
| `Details` | TEXT | Human-readable description |
| `Timestamp` | DATETIME | When it occurred |

#### `profiles`
| Column | Type | Description |
|---|---|---|
| `id` | INT (PK, Auto) | Unique profile identifier |
| `student_id` | INT (FK → students, Unique) | 1:1 with student |
| `profile_image_path` | VARCHAR(255) | Path to stored face image |
| `image_type` | ENUM('upload','webcam') | How the image was captured |
| `face_detected` | TINYINT(1) | Whether a face was found in the image |

---

## 🚀 Installation & Setup

### Prerequisites

- **Python 3.9+**
- **MySQL 5.7+** or **MariaDB 10.4+** (XAMPP, MAMP, or standalone)
- **Webcam** (for student-side proctoring)
- **Modern browser** (Chrome/Edge recommended for `getUserMedia`)

### Step-by-Step

#### 1. Clone the Repository

```bash
git clone https://github.com/h-990/online-cheat-detection-main.git
cd online-cheat-detection-main
```

#### 2. Create Virtual Environment

```bash
python -m venv .venv

# macOS/Linux
source .venv/bin/activate

# Windows
.venv\Scripts\activate
```

#### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

#### 4. Download YOLO Model Weights

```bash
python download_yolo_models.py
```

This downloads `yolov8n.pt` (~6MB) for real-time object detection.

#### 5. Setup Database

Start your MySQL/MariaDB server, then:

```bash
mysql -u root -p < examproctordb.sql
```

Or import `examproctordb.sql` through phpMyAdmin.

> **Note:** The SQL file creates the `examproctordb` database with all tables, indexes, and foreign key constraints.

#### 6. Configure Environment (Optional)

Create a `.env` file for custom settings:

```env
MYSQL_HOST=127.0.0.1
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=
MYSQL_DB=examproctordb
FLASK_SECRET_KEY=your-secret-key
```

#### 7. Run the Application

```bash
python app.py
```

The server starts at: **http://127.0.0.1:5001**

#### 8. Access the Application

| URL | Purpose |
|---|---|
| `http://127.0.0.1:5001/` | Landing page |
| `http://127.0.0.1:5001/login` | Login (admin or student) |
| `http://127.0.0.1:5001/signup` | Student registration |

> **⚠️ Important:** Camera access requires the page to be served from `localhost`, `127.0.0.1`, or via HTTPS.

### Default Admin Account

| Field | Value |
|---|---|
| Email | `admin@gmail.com` |
| Password | *(set in database — use the hashed password in the SQL dump)* |

---

## ⚙️ Configuration

All tunable parameters live in `config_vision.py`:

### Detection Thresholds

```python
YAW_THRESHOLD_DEG = 25         # Degrees of head turn before flagging
EAR_THRESHOLD = 0.18           # Eye Aspect Ratio below this = eyes closed
IRIS_OFFSET_THRESHOLD = 0.28   # Iris offset beyond this = looking away
```

### Temporal Debouncing

These control how long a condition must **persist continuously** before a warning fires:

```python
TIME_NO_FACE = 3.0             # Face must be missing for 3 full seconds
TIME_HEAD_TURNED = 2.0         # Head must be turned for 2 seconds
TIME_EYES_CLOSED = 1.0         # Eyes must be closed for 1 second
TIME_GAZING = 2.0              # Must be gazing away for 2 seconds
TIME_MULTIPLE_PERSONS = 1.5    # Multiple people for 1.5 seconds
TIME_BANNED_OBJECT = 1.5       # Banned object visible for 1.5 seconds
```

### Warning Aggregation

```python
WARNING_COOLDOWN_SEC = 5       # Seconds between repeated warnings of same type
INSTANT_PENALTY_THRESHOLD = 3  # If >= 3 rules broken simultaneously, bypass cooldown
```

### Model Configuration

```python
YOLO_MODEL_NAME = "yolov8n.pt"  # YOLOv8 nano for speed
YOLO_PERSON_CLASS = 0           # COCO class ID for 'person'
YOLO_BANNED_CLASSES = [63, 67, 73]  # laptop, cell phone, book
```

---

## 🎓 Student Exam Flow

The student goes through a carefully designed multi-step process:

```
Login → Rules → System Check → Face Registration → Face Confirmation → Exam → Result
```

### 1. Login (`login.html`)
- Email + password authentication
- Role-based routing (Admin → dashboard, Student → exam flow)
- Password reset via email link

### 2. Rules & Regulations (`ExamRules.html`)
- Displays exam conduct rules
- Student must acknowledge before proceeding

### 3. System Compatibility Check (`ExamSystemCheck.html`)
- **Camera:** Tests `getUserMedia` access
- **Microphone:** Tests audio input availability
- **Browser:** Verifies compatible browser features
- **Full-Screen:** Tests fullscreen API support
- Blocks exam start if any check fails

### 4. Face Registration (`ExamFaceInput.html`)
- Capture face via webcam or upload photo
- Face is stored in `static/Profiles/`
- Profile linked to student record in DB

### 5. Face Confirmation (`ExamConfirmFaceInput.html`)
- Captures live webcam snapshot
- Compares against registered face
- Prevents someone else from taking the exam

### 6. Exam (`Exam.html` — 150KB, full proctoring client)
- Randomized MCQ questions from `questions.js`
- Countdown timer with auto-submit
- **Background AI monitoring runs continuously:**
  - Webcam frames captured via `requestAnimationFrame` → sent to server
  - Microphone monitored via Web Audio API
  - Tab visibility tracked via `visibilitychange`
  - Fullscreen enforcement
- Visual warnings displayed on violation detection
- Termination screen shown if auto-terminated

### 7. Result (`ExamResult.html`, `ExamResultPass.html`, `ExamResultFail.html`)
- Score, percentage, grade displayed
- Confetti animation on pass 🎉
- Warning/violation summary

---

## 👨‍💼 Admin Dashboard

The admin dashboard (`Students.html`) is the central control center:

### Active Students Panel

- **Grid layout** showing all students currently in an exam
- Each student card displays:
  - Profile avatar
  - Student name and ID
  - Live monitoring status indicator (green dot)
  - **Clear Warnings** button — resets a student's warning count
  - **Force Terminate** button — immediately ends a student's exam
  - **View Live Feed** button — opens MJPEG video stream with AI overlays

### Auto-Terminate Toggle

A toggle switch at the top of the active students panel:

| State | Behavior |
|---|---|
| **OFF** (default) | Warnings accumulate, admin gets notifications, but exams are **never** auto-terminated |
| **ON** | After 3 warnings, the student's exam is automatically terminated |

### Real-Time Toast Notifications

The admin receives instant alerts via Socket.IO when any student:
- Triggers a violation (face not detected, etc.)
- Needs manual review
- Has their exam auto-terminated

### Additional Admin Pages

| Page | Purpose |
|---|---|
| **Results** (`Results.html`) | All exam results with filters |
| **Result Details** (`ResultDetails.html`) | Per-student breakdown with violation timeline |
| **Recordings** (`Recordings.html`) | Saved video recordings of violations |
| **Live Dashboard** (`admin_live_dashboard.html`) | Dedicated full-screen monitoring view |

---

## 🤖 AI Detection Pipeline

### Pipeline Architecture

```
Raw Frame (BGR)
    │
    ├──→ FaceAnalyzer (face_pipeline.py)
    │        │
    │        ├── MediaPipe FaceMesh (468 landmarks + 10 iris)
    │        ├── min_detection_confidence: 0.3
    │        ├── static_image_mode: True (thread-safe)
    │        │
    │        └── Returns:
    │              ├── face_detected (bool)
    │              ├── yaw_angle (float, degrees)
    │              ├── ear (float, Eye Aspect Ratio)
    │              ├── iris_offset_ratio (float)
    │              └── landmarks (478 points)
    │
    ├──→ PersonDetector (person_pipeline.py)
    │        │
    │        ├── YOLOv8 Nano (yolov8n.pt)
    │        ├── Classes detected: person(0), phone(67), book(73), laptop(77)
    │        │
    │        └── Returns:
    │              ├── person_count (int)
    │              ├── bounding_boxes [(x1,y1,x2,y2,conf), ...]
    │              └── banned_objects [{label, bbox}, ...]
    │
    └──→ DecisionEngine (decision_engine.py)
             │
             ├── Temporal debouncing per condition per student
             ├── Condition must persist for X seconds before warning
             │
             └── Returns:
                   ├── active_warnings ["Face not detected", ...]
                   └── penalty_score (float)
```

### Face Analysis Details

The `FaceAnalyzer` uses MediaPipe's 478-point face mesh to compute:

#### Yaw Angle (Head Turning)
```
eye_center_x = (left_eye.x + right_eye.x) / 2
offset = nose_tip.x - eye_center_x
yaw = (offset / face_width) / 0.20 * 25.0 degrees
```

#### Eye Aspect Ratio (EAR)
```
EAR = (vertical_dist_1 + vertical_dist_2) / (2 * horizontal_dist)
```
Low EAR (< 0.18) indicates closed eyes or looking down.

#### Iris Offset
```
iris_position = (iris_x - eye_corner_x) / eye_width - 0.5
```
High offset (> 0.28) indicates the student is looking sideways.

### Decision Engine: Temporal Debouncing

The decision engine prevents false positives by requiring conditions to **persist for a minimum duration** before triggering:

```python
def _check_condition(state, key, is_active, required_duration, now):
    if is_active:
        if state[key + '_start'] is None:
            state[key + '_start'] = now          # Start timer
        elif (now - state[key + '_start']) >= required_duration:
            return True                           # Condition held long enough → fire
    else:
        state[key + '_start'] = None              # Reset timer
    return False
```

---

## ⚠️ Warning & Violation System

### Warning Flow

```
AI Detection → DecisionEngine → WarningSystem → Socket.IO Events → Admin Dashboard
                                      │
                                      └──→ MySQL violations table
```

### Violation Types

| Type | Trigger |
|---|---|
| `NO_FACE` | Face not visible for 3+ seconds |
| `MULTIPLE_FACES` | More than one person detected *(currently disabled)* |
| `DISTRACTION` | Head turned or gazing away *(currently disabled)* |
| `EYES_CLOSED` | Eyes closed for extended period *(currently disabled)* |
| `TAB_SWITCH` | Student left the exam tab *(currently disabled)* |
| `VOICE_DETECTED` | Microphone detected voice/noise |
| `PROHIBITED_OBJECT` | Phone, book, or laptop detected *(currently disabled)* |
| `CAMERA_OFF` | Camera feed stopped *(currently disabled)* |
| `PROHIBITED_SHORTCUT` | Keyboard shortcut attempted *(currently disabled)* |

> **Current State:** Only `NO_FACE` detection is active. All other rules are commented out and can be re-enabled individually by uncommenting the corresponding code in `app.py` and `decision_engine.py`.

### Auto-Termination

When enabled by the admin toggle:
1. Each warning increments the student's counter
2. At **3 warnings**, the exam is automatically terminated
3. The student sees a termination screen with the reason
4. The exam auto-submits with an 'F' grade after 5 seconds
5. Camera access is preserved (not revoked) after termination

---

## 🔌 Real-Time Communication

### Socket.IO Namespaces

| Namespace | Users | Purpose |
|---|---|---|
| `/student` | Students | Send violations, receive warnings/termination |
| `/admin` | Administrators | Receive violation alerts, send control commands |

### Events: Student → Server

| Event | Data | Purpose |
|---|---|---|
| `tab_switch_detected` | `{student_id, details}` | Tab visibility change |
| `voice_detected` | `{student_id, details}` | Microphone activity |
| `prohibited_action` | `{student_id, action}` | Keyboard shortcut |
| `exam_auto_terminated` | `{student_id, reason, warnings}` | Exam ended |

### Events: Server → Admin

| Event | Data | Purpose |
|---|---|---|
| `student_violation` | `{student_id, type, details, count}` | New violation |
| `student_needs_review` | `{student_id, reason}` | Manual review needed |
| `student_exam_terminated` | `{student_id, reason}` | Exam terminated |
| `warnings_cleared` | `{student_id}` | Warnings reset |
| `enforcement_toggled` | `{enabled}` | Auto-terminate state |

### Events: Server → Student

| Event | Data | Purpose |
|---|---|---|
| `auto_terminated` | `{student_id}` | Terminate exam |
| `warnings_cleared` | `{student_id}` | Reset warning display |

---

## 📡 API Endpoints

### Authentication
| Method | Endpoint | Description |
|---|---|---|
| GET/POST | `/login` | User authentication |
| GET/POST | `/signup` | Student registration |
| GET | `/logout` | Session termination |
| POST | `/forgot-password` | Password reset request |
| POST | `/reset-password/<token>` | Password reset |

### Student Exam
| Method | Endpoint | Description |
|---|---|---|
| GET | `/exam-rules` | Exam rules page |
| GET | `/system-check` | System compatibility check |
| GET/POST | `/exam-face-input` | Face registration |
| GET/POST | `/exam-confirm-face` | Face verification |
| GET/POST | `/exam` | Main exam page / submission |

### Proctoring API
| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/student-frame` | Upload webcam frame for AI analysis |
| POST | `/api/student-exit-signal` | Beacon when student leaves tab |

### Admin API
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/admin-active-students` | List active exam students with status |
| GET | `/api/all-student-warnings` | Get all student warning counts |
| GET | `/admin/live/<student_id>` | MJPEG live video stream |
| GET | `/api/object-detection-status` | YOLO model status |
| GET | `/api/detection-pipeline-status` | Full pipeline status |

### Results
| Method | Endpoint | Description |
|---|---|---|
| GET | `/results` | All exam results |
| GET | `/result-details/<student_id>` | Per-student results |
| GET | `/api/today-violations` | Today's violation summary |

---

## 🔒 Security Features

| Feature | Implementation |
|---|---|
| **Password Hashing** | bcrypt / pbkdf2-sha256 with salt |
| **Session Management** | Flask sessions with HTTP-only cookies |
| **CSRF Protection** | Token validation on all POST requests |
| **Role-Based Access** | `@require_role('ADMIN')` decorator on admin routes |
| **Camera Permissions** | Requires localhost or HTTPS |
| **Input Sanitization** | Length limits and type validation on all inputs |
| **Secure Cookies** | `SameSite=Lax`, `HTTPOnly=True` |

---

## 🔧 Troubleshooting

### Camera Not Working

- **Cause:** Browser blocks `getUserMedia` on non-secure origins
- **Fix:** Access via `http://127.0.0.1:5001` or `http://localhost:5001`, NOT via IP

### "No Face Detected" False Positives

- **Cause:** Poor lighting or webcam angle
- **Fix:** Ensure adequate lighting on your face, face the camera directly
- **Tuning:** Increase `TIME_NO_FACE` in `config_vision.py` (currently 3.0s)
- **Tuning:** Lower `min_detection_confidence` in `face_pipeline.py` (currently 0.3)

### Admin Dashboard Empty

- **Cause:** No students currently in an exam
- **Fix:** The dashboard auto-populates when students start exams

### MySQL Connection Refused

- **Cause:** MySQL/MariaDB not running or wrong credentials
- **Fix:** Start MySQL service, verify credentials in `.env` or `app.py`

### YOLO Model Missing

- **Cause:** `yolov8n.pt` not downloaded
- **Fix:** Run `python download_yolo_models.py`

### Socket.IO Disconnections

- **Cause:** Eventlet async issues or firewall
- **Fix:** Ensure `eventlet` is installed, check no proxy is blocking WebSocket

---

## 📄 License

This project is for educational purposes.

---

<div align="center">

**Built with ❤️ using Python, Flask, OpenCV, MediaPipe, and YOLOv8**

</div>
]]>
