# IDVRS (Bhu-Setu) — Intelligent Land Record Digitization & Validation System

A full-stack, AI-powered system designed to digitize, extract, validate, and spatially link legacy land records (such as **Khatoni / Form-7**, **Bhu-Adhikar Pustika / Form-4**, historic handwritten registers, and English revenue deeds) into structured databases.

---

## 📌 Overview

Land records across Indian states often exist as legacy physical registers, scanned photocopies, handwritten patwari notes, or certified digital PDFs. Manually transcribing these documents into state land registries is slow, expensive, and error-prone.

**IDVRS (Bhu-Setu)** streamlines this process through an end-to-end pipeline:
1. **Batch Upload**: Upload multiple PDFs or images at once with queue tracking.
2. **Multi-Tiered Extraction**: Direct text parsing for native PDFs, local OCR for printed scans, and Multimodal Vision for unconstrained handwriting.
3. **Automated Validation**: Instant verification of duplicate records, fractional ownership share summation, and format checks.
4. **Human-in-the-Loop (HITL) Review**: Side-by-side inspection console with interactive editing, field-level confidence scores, and a sequential review queue.
5. **GIS Integration**: Cross-matching digitized land records with cadastral survey parcel polygons on interactive maps with satellite imagery.
6. **Registry & Export**: Generates printable **Record of Rights (RoR)** certificates, CSV exports, and central dashboard analytics.

---

## 🚀 Key Features & Capabilities

### 1. Multi-Document Batch Upload Studio (`/upload`)
- **Queue Management**: Drag-and-drop or select multiple documents (`.pdf`, `.jpg`, `.jpeg`, `.png`) simultaneously.
- **Live Processing Progress**: Real-time status indicators per document (`pending` → `processing` → `completed` / `error`).
- **1-Click Test Presets**: Pre-loaded samples for immediate testing:
  - *Certified Copy Form-4* (Bhu-Adhikar Pustika)
  - *Certified Copy Form-7* (Khatoni B1)
  - *Handwritten Register* (Historic Simariya / Panna land register)
  - *English Land Record* (Revenue allotment deed)
  - *Load All Batch*: 1-click loading of all evaluation documents into the queue.

### 2. Multi-Tiered Extraction Pipeline
- **Tier 1: Direct Digital PDF Parser**:
  - Uses `PyMuPDF` (`fitz`) and `pdfplumber` to extract clean text from portal-generated PDFs in under 50ms without OCR artifacts.
- **Tier 2: Scanned OCR Engine**:
  - OpenCV image preprocessing (adaptive thresholding, noise removal, grayscale conversion).
  - Tesseract OCR engine configured for bilingual Hindi (`hin`) and English (`eng`).
- **Tier 3: Neural Vision Adapter (`gemini_vision_adapter`)**:
  - Leverages Google Gemini Multimodal Vision API for complex multi-column layouts, degraded ink, and handwritten cadastral registers.
- **Devanagari Normalization & Heuristic Extraction**:
  - Unicode NFC normalization and matra deduplication (e.g., resolving rendering quirks like `पन्नाा` → `पन्ना`).
  - Regex and rule-based extractors for Khata numbers, CLRM IDs, villages, tehsils, co-owners, survey/khasra numbers, land uses, and areas in hectares.

### 3. Mathematical & Legal Validation Engine (`validator.py`)
- **Duplicate Detection**: Real-time database checks for duplicate Khata numbers and CLRM identifiers within the same village/tehsil.
- **Ownership Share Balancing**: Automatically converts fractional shares (e.g., $1/2 + 1/4 + 1/4$) and checks whether they sum to $1.0$ ($100\%$).
- **Mandatory Field Audits**: Flags missing critical attributes (village, district, survey number, area) before records can be committed.
- **Authoritative Server-Side Validation**: All client submissions undergo strict server-side re-validation before database persistence.

### 4. Human-in-the-Loop (HITL) Review Console (`/review`)
- **Dual-Pane Interface**: Original document viewer (zoom in/out, 90° rotation, full-screen) side-by-side with editable form fields.
- **Dynamic Confidence Scoring**: Color-coded confidence badges per field (Green $\ge 85\%$, Amber $60-84\%$, Red $< 60\%$).
- **Editable Owners & Parcels Tables**: Add, edit, or delete co-owners and survey parcels inline.
- **Sequential Batch Review**: When reviewing a batch, saving one document automatically loads the next document in the queue with a status counter banner (*Document 1 of 4*).

### 5. GIS Cadastral Map View (`/gis`)
- **Interactive Cadastral Viewer**: Built with Leaflet, visualizing cadastral survey parcel polygons.
- **Multi-Layer Support**:
  - OpenStreetMap (Standard base)
  - CartoDB Voyager (Clean cartography)
  - Google Satellite Hybrid (`mt1.google.com` high-resolution satellite imagery with road overlays)
- **Spatial Cross-Matching**: Runtime matching between spatial parcels and relational MySQL records:
  - 🟢 **Matched**: Parcel boundary matches an active digitized record.
  - ⚪ **Unmatched**: Spatial parcel exists without a corresponding database record.
  - 🟠 **Discrepant**: Area mismatch between spatial polygon and revenue document.
- **Administrative Hierarchy Filter**: Drill down by State → District → Tehsil → Village.

### 6. Official RoR Certificate & Central Registry
- **Record of Rights Certificate (`/records/:id`)**: Official revenue format with auto-calculated percentage shares, barcode, stamp watermarks, and print CSS optimization (`Ctrl + P`).
- **Registry Management (`/records`)**: Full text search, status filters (All, Verified, Pending, Duplicate), test record deletion, and 1-click CSV export.
- **Executive Analytics (`/`)**: KPI counters, district-wise distribution bar chart, and validation status breakdown.
- **Bilingual Interface**: Seamless 1-click toggle between **हिन्दी (Hindi)** and **English** across the entire UI.

---

## 🏗️ System Architecture

```text
┌────────────────────────────────────────────────────────────────────────┐
│                   Frontend Client (React 19 + Vite)                     │
│  • Dashboard   • Batch Upload Studio   • HITL Review   • GIS Map Viewer │
│  • Records Registry   • Printable RoR Certificate   • Bilingual (Hi/En) │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ REST API (/api/*)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   Backend Service (FastAPI / Uvicorn)                   │
│                                                                        │
│  ┌────────────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ Document Type Detector │  │ GIS Matching    │  │ Record & Export │ │
│  │ (Digital / Scan / HWR) │  │ Engine (/gis)   │  │ Handlers        │ │
│  └───────────┬────────────┘  └─────────────────┘  └─────────────────┘ │
│              │                                                         │
│              ▼                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Extraction Tier Router:                                          │  │
│  │  1. PyMuPDF / pdfplumber (Native PDFs)                           │  │
│  │  2. OpenCV + Tesseract (Standard Scans - hin+eng)                │  │
│  │  3. Gemini Multimodal Vision Adapter (Handwritten / Unstructured)│  │
│  └───────────────────────────┬──────────────────────────────────────┘  │
│                              │                                         │
│                              ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ Normalization & Validation:                                      │  │
│  │  • Devanagari NFC & Matra Cleanup   • Field Extraction           │
│  │  • Duplicate Record Detection       • Share Summation Validation │  │
│  └───────────────────────────┬──────────────────────────────────────┘  │
└──────────────────────────────┼─────────────────────────────────────────┘
                               │
                               ▼
┌────────────────────────────────────────────────────────────────────────┐
│                     Relational Database (MySQL)                        │
│  • documents   • khatas   • khata_owners   • khata_parcels             │
│  (Supports Local MySQL 8.0 & Cloud TiDB Serverless with SSL)           │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 💻 Tech Stack

| Component | Technology | Purpose |
|---|---|---|
| **Frontend Framework** | React 19, Vite, React Router 7 | Responsive, high-speed single page application |
| **Mapping & GIS** | Leaflet, React-Leaflet | Cadastral parcel visualization and satellite tile rendering |
| **Icons & Charts** | Lucide React, Recharts | UI iconography and executive KPI visual analytics |
| **Internationalization** | i18next, react-i18next | Full English and Hindi bilingual localization |
| **Backend API** | Python 3.10+, FastAPI, Uvicorn | Asynchronous REST backend with automatic OpenAPI docs |
| **PDF Extraction** | PyMuPDF (`fitz`), `pdfplumber` | Direct digital text and table extraction |
| **Computer Vision / OCR**| OpenCV (headless), Tesseract OCR | Image enhancement, binarization, deskewing, and OCR |
| **Neural Vision** | Google Gemini API (`gemini-3.1-flash-lite`) | Multimodal extraction for handwritten and complex registers |
| **Validation** | Pydantic v2, Python standard library | Strict payload validation and cross-field mathematical checks |
| **Database ORM** | SQLAlchemy 2.0, PyMySQL | Relational data models, foreign keys, and connection pooling |
| **Database Engine** | MySQL 8.0 / TiDB Cloud (Serverless) | Relational storage with UTF8mb4 Devanagari charset and SSL |

---

## 📁 Repository Structure

```text
SIH-Prototype_IDVRS/
├── Backend/
│   ├── api/
│   │   └── index.py               # Vercel serverless entrypoint
│   ├── app/
│   │   ├── api/
│   │   │   ├── dashboard.py       # Metrics, analytics, and status endpoints
│   │   │   ├── gis.py             # GIS cadastral parcels, hierarchy, cross-matching
│   │   │   ├── records.py         # Save record, delete, purge, and fetch
│   │   │   └── upload.py          # Single and batch upload pipeline endpoints
│   │   ├── pipeline/
│   │   │   ├── detector.py        # Native PDF vs scanned classifier
│   │   │   ├── extractor.py       # PyMuPDF digital parser & Tesseract OCR
│   │   │   ├── field_extractor.py #  extraction & Devanagari normalization
│   │   │   ├── gemini_vision_adapter.py # Multimodal vision for handwritten docs
│   │   │   ├── handwriting_adapter.py  # Indic EasyOCR & fallback adapter
│   │   │   ├── handwriting_detector.py # Script/handwriting classification
│   │   │   ├── preprocessor.py    # OpenCV grayscale, thresholding & denoise
│   │   │   └── validator.py       # Duplicates, share fraction & format checks
│   │   ├── config.py              # Environment configuration & DB connection string
│   │   ├── database.py            # SQLAlchemy engine, session maker & TiDB SSL
│   │   ├── main.py                # FastAPI app initialization, routes & CORS
│   │   └── models.py              # SQLAlchemy schema (documents, khatas, owners, parcels)
│   ├── gis_data/
│   │   └── cadastral_demo.geojson # Spatial cadastral boundary dataset
│   ├── uploads/                   # Runtime temporary file uploads
│   ├── .env.example               # Environment variables template
│   ├── requirements.txt           # Python backend dependencies
│   └── vercel.json                # Backend Vercel deployment configuration
│
├── Frontend/
│   ├── public/
│   │   └── samples/               # Bundled sample PDFs and images for quick testing
│   ├── src/
│   │   ├── components/
│   │   │   └── ConfidenceBadge.jsx# Visual confidence indicators
│   │   ├── i18n/                  # English and Hindi translation dictionaries
│   │   │   ├── en.json
│   │   │   └── hi.json
│   │   ├── pages/
│   │   │   ├── DashboardPage.jsx  # Executive KPIs and analytics
│   │   │   ├── GISPage.jsx        # Cadastral map with satellite hybrid layer
│   │   │   ├── RecordDetailPage.jsx # Printable RoR certificate
│   │   │   ├── RecordsPage.jsx    # Records registry, search, CSV export
│   │   │   ├── ReviewPage.jsx     # HITL split-pane review & sequential queue
│   │   │   └── UploadPage.jsx     # Batch Upload Studio & queue manager
│   │   ├── App.jsx                # Layout, navigation bar, and routing
│   │   ├── index.css              # Styling tokens, responsive grid & print styles
│   │   └── main.jsx               # React entrypoint
│   ├── package.json               # Node frontend dependencies
│   ├── vercel.json                # Frontend Vercel rewrites & API proxy
│   └── vite.config.js             # Vite configuration and local proxy
│
└── README.md                      # Comprehensive project documentation
```

---

## ⚙️ Local Setup & Installation

### Prerequisites
1. **Python 3.10+**: [python.org](https://www.python.org/downloads/) *(Ensure "Add Python to PATH" is checked)*
2. **Node.js 18+ & npm**: [nodejs.org](https://nodejs.org/)
3. **MySQL Server 8.0+** (or Cloud MySQL / TiDB Cloud instance)
4. *(Optional)* **Tesseract OCR**: [Tesseract Windows Installer](https://github.com/UB-Mannheim/tesseract/wiki) *(select Hindi during installation)*

---

### Step 1: Clone the Repository
```bash
git clone https://github.com/shatansu/IDVRS.git
cd IDVRS
```

---

### Step 2: Database Setup
In your MySQL terminal, Workbench, or cloud database console, create the database:
```sql
CREATE DATABASE IF NOT EXISTS land_record_db 
CHARACTER SET utf8mb4 
COLLATE utf8mb4_unicode_ci;
```
*(All tables: `documents`, `khatas`, `khata_owners`, `khata_parcels` are automatically created on first backend startup).*

---

### Step 3: Backend Setup
1. Navigate to `Backend`:
   ```bash
   cd Backend
   ```
2. Create and activate a virtual environment:
   - **Windows (PowerShell)**:
     ```powershell
     python -m venv venv
     .\venv\Scripts\Activate.ps1
     ```
   - **Linux / macOS**:
     ```bash
     python3 -m venv venv
     source venv/bin/activate
     ```
3. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
4. Configure environment variables:
   Copy `.env.example` to `.env` and fill in your credentials:
   ```ini
   DB_HOST=localhost
   DB_PORT=3306
   DB_USER=root
   DB_PASSWORD=your_password
   DB_NAME=land_record_db
   API_HOST=127.0.0.1
   API_PORT=8000
   CORS_ORIGINS=http://localhost:5173,http://127.0.0.1:5173
   
   # Optional / Multimodal Vision
   GEMINI_API_KEY=your_gemini_api_key
   GEMINI_MODEL=gemini-3.1-flash-lite
   ```
   *(For Cloud TiDB/MySQL, you can alternatively set `DATABASE_URL=mysql+pymysql://...`)*

5. Run the backend:
   ```bash
   uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
   ```
   Interactive Swagger docs will be accessible at: `http://127.0.0.1:8000/docs`

---

### Step 4: Frontend Setup
1. Open a new terminal window and navigate to `Frontend`:
   ```bash
   cd Frontend
   ```
2. Install dependencies:
   ```bash
   npm install
   ```
3. Run the development server:
   ```bash
   npm run dev
   ```
4. Access the web interface at:
   👉 **`http://localhost:5173`**

---

## 🌐 Live Deployment Architecture

The system is configured for cloud deployment across the following services:

| Component | Platform | Configuration | Notes |
|---|---|---|---|
| **Frontend** | **Vercel** | `Frontend/vercel.json` | Reverse proxies `/api/*` requests to the live backend |
| **Backend** | **Render** | Web Service (`bhu-setu-backend.onrender.com`) | Handles long-running OCR, vision pipeline, and file uploads |
| **Database** | **TiDB Cloud** | Serverless MySQL with SSL (`ssl_mode: REQUIRED`) | Managed cloud relational storage with high availability |
| **CI/CD** | **GitHub** | `https://github.com/shatansu/IDVRS.git` | Auto-deploys from connected branches |

---

## 📡 API Reference Summary

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/upload` | Uploads and processes a single land record document |
| `POST` | `/api/upload/batch` | Uploads and processes multiple documents in batch mode |
| `POST` | `/api/records` | Persists verified land record with server-side validation |
| `GET` | `/api/records` | Retrieves paginated list of saved land records |
| `GET` | `/api/records/{id}` | Fetches detailed record data (owners, parcels, validation) |
| `DELETE` | `/api/records/{id}` | Deletes a single record and cascades to owners/parcels |
| `DELETE` | `/api/records?confirm=true` | Purges all records and documents (testing reset) |
| `GET` | `/api/dashboard/stats` | Returns aggregate counts, confidence metrics, and charts |
| `GET` | `/api/gis/parcels` | Returns cadastral GeoJSON features enriched with matching status |
| `GET` | `/api/gis/hierarchy` | Returns administrative geographic hierarchy (State/District/Tehsil/Village) |
| `GET` | `/api/gis/khata/{id}` | Fetches linked spatial features for a specific Khata |

---

## 📜 Standards & Compliance
Designed aligned with the objectives of the **Digital India Land Records Modernization Programme (DILRMP)** and the guidelines of the **Department of Land Resources (DoLR), Ministry of Rural Development, Government of India**.
