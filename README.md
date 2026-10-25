# 🛣️ RoadWatch

**AI-powered road damage reporting system.**
Citizens photograph road damage from their phone; municipal staff review, prioritize, and track reports on a web dashboard.

![Status](https://img.shields.io/badge/status-in%20development-orange)
![Flutter](https://img.shields.io/badge/mobile-Flutter-02569B?logo=flutter&logoColor=white)
![FastAPI](https://img.shields.io/badge/backend-FastAPI-009688?logo=fastapi&logoColor=white)
![YOLOv8](https://img.shields.io/badge/AI-YOLOv8n-00FFFF)
![PostGIS](https://img.shields.io/badge/database-PostgreSQL%20%2B%20PostGIS-336791?logo=postgresql&logoColor=white)

---

## 📌 Overview

Road damage is often reported slowly, inconsistently, and without structured data. RoadWatch turns a single photo into a structured, location-tagged, prioritized report.

**Core pipeline:**

```
Photo → AI detection → Severity estimate → Location-tagged report
      → Duplicate grouping → Prioritized staff review → Status tracking
```

The system has two interfaces connected to one shared backend:

| Interface | Users | Purpose |
|---|---|---|
| **Flutter mobile app** | Citizens | Capture damage, get an AI category and preliminary severity, attach GPS and a description, submit |
| **Web admin dashboard** | Municipal staff | View reports by severity, filter, inspect potential duplicates, update status |

---

## ✨ Features

### Citizen app
- Capture a photo with the camera or pick one from the gallery
- GPS location with permission handling (granted, denied, services off)
- AI-detected damage category, confidence, and preliminary severity
- Optional text description
- Submission confirmation with a report ID
- Graceful handling of failed uploads, unreadable images, no network, and no detection

### Admin dashboard
- Reports listed by severity (highest first, ties broken by newest)
- Filter by damage category and status
- Report detail: photo, category, severity, GPS, description, date, status
- Potential duplicate groups with originals individually accessible
- Status updates: `Submitted → Under Review → Resolved`

### Backend
- Input validation (file type, size, coordinate ranges)
- YOLOv8n inference loaded once at startup
- Rule-based severity scoring
- DBSCAN geographic clustering (originals are never deleted or merged)
- Staff-only endpoints protected by authentication

---

## 🧠 AI Components

| Component | Approach |
|---|---|
| **Damage detection** | YOLOv8n (Ultralytics), fine-tuned on RDD2022 via transfer learning |
| **Severity estimation** | Rule-based score from category weight, bounding-box area fraction, and confidence → `Low / Medium / High` |
| **Duplicate grouping** | DBSCAN with haversine distance on report coordinates |

> ⚠️ **Responsible AI:** Severity is a *preliminary, image-based indicator*, not a professional engineering assessment. Clusters are *suggestions* for human review. Staff always have the final say.

---

## 🏗️ Architecture

```
┌──────────────┐        ┌────────────────────────────────────┐
│ Flutter App  │─HTTPS─▶│          FastAPI Backend           │
└──────────────┘        │  • request validation              │
                        │  • YOLOv8n inference               │
┌──────────────┐        │  • severity scoring                │
│ Web Dashboard│─HTTPS─▶│  • DBSCAN clustering               │
└──────────────┘        │  • staff auth, status updates      │
                        └──────────┬───────────────┬─────────┘
                                   │               │
                        ┌──────────▼─────┐  ┌──────▼───────────┐
                        │ PostgreSQL +   │  │ Object storage   │
                        │ PostGIS        │  │ (report photos)  │
                        └────────────────┘  └──────────────────┘
```

**Design principles**
- Synchronous processing (no job queue in the MVP)
- Images live in object storage; the database stores references only
- The backend is the single source of truth, so clients never compute severity or clusters

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Mobile app | Flutter |
| Backend API | Python, FastAPI |
| Computer vision | Ultralytics YOLOv8n |
| Clustering | scikit-learn (DBSCAN) |
| Database | PostgreSQL + PostGIS (Supabase) |
| Image storage | Supabase Storage |
| Deployment | Render (or equivalent) |
| Version control | Git + GitHub |

---

## 📁 Repository Structure

> Proposed layout. Update it as the project evolves.

```
roadwatch/
├── mobile/          # Flutter app
├── backend/         # FastAPI service
├── ml/              # Dataset prep, training, evaluation, inference module
├── modules/         # Severity scoring and DBSCAN clustering
├── dashboard/       # Web admin dashboard
├── database/        # Schema and migrations
├── docs/            # API contract, severity rules, evaluation reports
├── .env.example
└── README.md
```

---

## 🔌 API Overview

All endpoints are versioned under `/api/v1`. Full docs are available at `/docs` (FastAPI OpenAPI).

| Method | Endpoint | Purpose | Auth |
|---|---|---|---|
| `POST` | `/analyze` | Upload image → detections and severity | Public |
| `POST` | `/reports` | Submit a report (image, lat, lon, description) | Public |
| `GET` | `/reports` | List reports (sort, category, status, pagination) | Staff |
| `GET` | `/reports/{id}` | Report detail with cluster mates | Staff |
| `PATCH` | `/reports/{id}/status` | Update report status | Staff |
| `GET` | `/clusters` | Duplicate groupings | Staff |
| `GET` | `/health` | Health check and model version | Public |

---

## 🚀 Getting Started

> Setup instructions will be filled in as each component is completed.

### Prerequisites
- Python 3.10+
- Flutter SDK
- A Supabase project (PostgreSQL/PostGIS + Storage)

### Environment variables

Copy `.env.example` to `.env` and fill in your own values. **Never commit `.env`.**

### Backend
```bash
cd backend
python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

### Mobile app
```bash
cd mobile
flutter pub get
flutter run
```

### Dashboard
_Instructions to be added._

---

## 🗺️ Roadmap

| Week | Focus | Status |
|---|---|---|
| 1 | Requirements, API contract, DB schema, dataset prep | ⬜ |
| 2 | Baseline model and reporting pipeline | ⬜ |
| 3 | End-to-end AI integration | ⬜ |
| 4 | Severity ordering, duplicates, status management | ⬜ |
| 5 | Evaluation and deployment | ⬜ |
| 6 | Validation and final presentation | ⬜ |

**Optional extensions (after MVP is stable):** Leaflet map view, own-report history, image-quality checks, role-based staff access.

---

## 📊 Evaluation

Results will be reported honestly, including failures.

- **Detector:** mAP@0.5, per-class precision and recall, inference time (on a held-out test split)
- **Clustering:** hand-labeled coordinate test sets (close groups, separated groups, isolated points, boundary cases)
- **Severity:** unit tests on the documented rules
- **End-to-end:** full workflow on locally collected photos not used in training

_Results table to be added after Week 5._

---

## ⚠️ Limitations

- Severity is a preliminary indicator, not an engineering assessment
- Camera angle, distance, and lighting affect bounding-box measurements
- Geographic proximity does not prove two reports describe the same damage
- The detector is limited to its training classes and may perform differently on local roads
- GPS accuracy varies, especially in dense urban areas
- Authenticity of citizen reports is not verified

---

## 🔒 Privacy

Faces and license plates are blurred in any locally collected photos used in documentation or demos. Citizen-submitted images are not published.

---

## 👥 Team

| Role | Member |
|---|---|
| Mobile Application | _Name_ |
| Backend and Security | _Name_ |
| Database and Geospatial Data | _Name_ |
| Computer Vision | _Name_ |
| Severity and Duplicate Detection | _Name_ |
| Admin Dashboard and Deployment | _Name_ |

---

## 📄 License and Third-Party Notes

_License to be decided by the team._

This project uses [Ultralytics YOLOv8](https://github.com/ultralytics/ultralytics), which is licensed under **AGPL-3.0** (with an enterprise option). Check the terms before any commercial use. Training data comes from **RDD2022**; verify its license and class definitions before redistribution.
