# MetraCheck — Step-by-Step Implementation Plan

**Companion document to:** PRD, TRD, App Flow, UI/UX Brief, Backend Schema (v1.0)

---

## 1. Phased Sprint Timeline

### Phase 1 — Environment Setup, DB Schemas, & React/Flask Boilerplate

**Goal:** A working skeleton where a user can log in, and the frontend and backend can talk to each other, with the databases provisioned.

- Set up monorepo or two-repo structure (`metracheck-web/`, `metracheck-backend/`).
- Provision PostgreSQL and MongoDB instances (local Docker for dev; managed services for staging/prod).
- Implement `users`, `legal_rules_config`, `inspections_master`, `official_notices` tables via migrations (Alembic).
- Implement JWT auth endpoints (`/auth/login`, `/auth/refresh`, `/auth/logout`) and RBAC middleware.
- Scaffold React app with routing (React Router), Tailwind CSS setup, and role-based route guards matching the Information Architecture Map.
- Seed `legal_rules_config` with initial Rule 6–9 threshold values from the LMPC Rules, 2011.
- Set up Docker Compose for local dev (Flask API, Postgres, MongoDB, Redis).

**Milestone Acceptance Criteria:**
- A seeded Inspector, Supervisor, and Admin user can each log in and are routed to their correct landing page.
- RBAC middleware correctly rejects a cross-role request (e.g., Inspector hitting `/admin/rules-config`) with 403.
- `legal_rules_config` table is populated and readable via an internal API endpoint.

---

### Phase 2 — WebRTC Scanner & OpenCV Image Preprocessing / OCR Microservice

**Goal:** An inspector can capture or upload an image, and the backend returns raw OCR + barcode extraction results.

- Build the WebRTC capture component (live camera, framing overlay, capture button, thumbnail strip) per the UI/UX Brief §4.2.
- Implement fallback file-upload path for devices/browsers without camera access.
- Build the image ingestion API endpoint: accepts image(s), computes SHA-256 hash, stores original in S3/Cloudinary, persists metadata to `inspections_master` with `overall_status='PROCESSING'`.
- Implement the OpenCV preprocessing pipeline: deskew, perspective correction, contrast enhancement (CLAHE).
- Integrate Tesseract OCR with bounding-box output; implement Google Cloud Vision fallback path for low-confidence results.
- Integrate PyZbar/ZXing for barcode decoding (EAN-13, GS1 DataMatrix, QR).
- Persist raw OCR + barcode output to MongoDB `unparsed_ocr_dumps`.
- Implement local capture queueing (IndexedDB) with auto-sync on reconnect for offline tolerance.

**Milestone Acceptance Criteria:**
- Capturing/uploading a sample product image returns a populated `unparsed_ocr_dumps` document with text blocks, bounding boxes, and any decoded barcode.
- A deliberately blurred/low-quality test image correctly falls back to Google Cloud Vision (or is flagged low-confidence).
- Offline capture queue correctly syncs a queued image once connectivity is restored (manually tested by toggling network).

---

### Phase 3 — Rule Validation Engine & Spatial Font Measurement Implementation

**Goal:** A processed image produces a full, rule-by-rule compliance verdict, including calibrated font-height measurement.

- Implement text-block classification (regex + keyword + positional heuristics) mapping OCR blocks to declaration categories (MRP, Net Qty, Mfg Date, Address, Consumer Care, FSSAI No.).
- Implement the spatial calibration engine: reference-object detection, package-relative fallback, and manual two-point calibration API + UI.
- Implement the font-height-to-mm computation per TRD §3.3.
- Implement the deterministic rule engine module evaluating Rules 6, 7, 8, 9 against `legal_rules_config`, producing the `rule_verdicts` array structure defined in the Backend Schema doc.
- Persist the full evaluation to MongoDB `compliance_audit_reports`; update `inspections_master.overall_status`.
- Build the Rule Inspection Card UI (per UI/UX Brief §4.4) rendering each verdict with expected/observed values and bounding-box thumbnail overlay.
- Implement the manual override workflow (linked record, not overwrite) on both backend and UI.

**Milestone Acceptance Criteria:**
- A test image with a known, deliberately undersized MRP declaration is correctly flagged FAIL on Rule 7 with a plausible measured mm value.
- A test image with all declarations present and compliant returns overall_status = COMPLIANT.
- A manual override submitted by a test Inspector account is stored as a separate linked record and both original + override are visible in the case detail view.

---

### Phase 4 — Verification Integrations, Sentiment Analysis, & ReportLab PDF Engine

**Goal:** External registry checks, advisory sentiment scoring, and a complete downloadable evidence report are functional.

- Implement GS1 GEPIR lookup integration (with caching in `master_registry_cache` and graceful timeout handling).
- Implement FSSAI FoSCoS lookup integration (same resilience pattern).
- Implement the RoBERTa-based sentiment/risk scoring pipeline over sample e-commerce review data, writing to MongoDB `product_risk_scores`, clearly separated from statutory verdicts.
- Implement the ReportLab/PDFKit report generator producing the full PDF (source images, extracted declarations, rule verdicts, overall status, inspector identity, SHA-256 hash, timestamp).
- Implement the Celery + Redis async task queue for e-commerce batch crawling (Playwright/Scrapy-based ingestion of listing images at scale) and the Auditor batch-triage UI (Flow C).
- Wire the "Route to Supervisory Officer" handoff from both the Field Inspector flow and the Auditor batch triage flow into a common case queue.

**Milestone Acceptance Criteria:**
- A test inspection with a valid GTIN returns a resolved GS1 entity name; a test inspection with an unreachable registry gracefully returns "unavailable" without failing the inspection.
- A generated PDF report opens correctly, includes all required sections, and its stored SHA-256 hash matches a recomputation of the original evidence image.
- A batch crawl job of at least 10 sample listing images completes asynchronously and produces a triage-able results list.

---

### Phase 5 — UI/UX Dashboard Polish, RBAC, End-to-End Testing, & Deployment

**Goal:** Production-ready system with full dashboard, hardened RBAC, tested end-to-end flows, and a deployed environment.

- Build the Supervisory Officer Dashboard (KPI tiles, violation-type breakdown chart, geographic heatmap) per PRD Epic 6.
- Build the searchable Repository view with filters (product, brand, inspector, region, status).
- Build the System Administrator console: user management, `legal_rules_config` editor UI, audit log viewer, system/queue health view.
- Full RBAC audit across every endpoint against the matrix in the Backend Schema doc §4.1.
- End-to-end test pass across all three flows (A: Inspector capture-to-report, B: Supervisor review-to-notice, C: E-commerce batch audit) including the edge cases documented in the App Flow doc §3.
- Performance pass against the design targets in the PRD (§5.1).
- Containerize with Docker; configure Nginx reverse proxy + Gunicorn; set up staging and production environments.
- Prepare demo dataset and walkthrough script for the SIH evaluation.

**Milestone Acceptance Criteria:**
- All five edge-case flows (App Flow §3.1–3.5) behave as specified when manually triggered.
- A Supervisory Officer can complete the full review-to-notice flow (Flow B) end-to-end against a real inspection created in Phase 3–4.
- Deployed staging environment is reachable over HTTPS and passes a smoke test of login + one full inspection cycle for each of the four roles.

---

## 2. Directory Tree Structure

### 2.1 Frontend — `metracheck-web/`

```
metracheck-web/
├── public/
│   └── index.html
├── src/
│   ├── api/
│   │   ├── authApi.js
│   │   ├── inspectionApi.js
│   │   ├── ruleConfigApi.js
│   │   └── batchApi.js
│   ├── components/
│   │   ├── common/
│   │   │   ├── Button.jsx
│   │   │   ├── StatusBadge.jsx
│   │   │   ├── Modal.jsx
│   │   │   └── DataTable.jsx
│   │   ├── scanner/
│   │   │   ├── WebRTCViewport.jsx
│   │   │   ├── FramingOverlay.jsx
│   │   │   └── CaptureThumbnailStrip.jsx
│   │   ├── inspection/
│   │   │   ├── RuleInspectionCard.jsx
│   │   │   ├── ExtractionReviewForm.jsx
│   │   │   └── OverrideModal.jsx
│   │   └── dashboard/
│   │       ├── KpiTile.jsx
│   │       ├── ViolationChart.jsx
│   │       └── HeatmapView.jsx
│   ├── pages/
│   │   ├── Login.jsx
│   │   ├── inspector/
│   │   │   ├── InspectorHome.jsx
│   │   │   ├── ScanPage.jsx
│   │   │   ├── ReviewPage.jsx
│   │   │   ├── VerdictPage.jsx
│   │   │   ├── EvidencePage.jsx
│   │   │   ├── ReportPage.jsx
│   │   │   └── InspectorHistory.jsx
│   │   ├── supervisor/
│   │   │   ├── SupervisorDashboard.jsx
│   │   │   ├── HeatmapPage.jsx
│   │   │   ├── InspectionListPage.jsx
│   │   │   ├── InspectionDetailPage.jsx
│   │   │   └── NoticePage.jsx
│   │   ├── auditor/
│   │   │   ├── BatchListPage.jsx
│   │   │   ├── NewBatchPage.jsx
│   │   │   ├── BatchResultsPage.jsx
│   │   │   └── ListingDetailPage.jsx
│   │   ├── admin/
│   │   │   ├── UserManagementPage.jsx
│   │   │   ├── RulesConfigPage.jsx
│   │   │   ├── AuditLogPage.jsx
│   │   │   └── SystemHealthPage.jsx
│   │   └── repository/
│   │       └── RepositorySearchPage.jsx
│   ├── context/
│   │   ├── AuthContext.jsx
│   │   └── RoleGuard.jsx
│   ├── hooks/
│   │   ├── useOfflineQueue.js
│   │   └── useCaptureQuality.js
│   ├── styles/
│   │   └── tailwind.css
│   ├── App.jsx
│   └── index.js
├── package.json
└── tailwind.config.js
```

### 2.2 Backend — `metracheck-backend/`

```
metracheck-backend/
├── app/
│   ├── __init__.py                 (Flask app factory)
│   ├── config.py
│   ├── extensions.py                (db, jwt, celery init)
│   ├── auth/
│   │   ├── routes.py
│   │   ├── models.py                (users - SQLAlchemy)
│   │   └── rbac.py                  (role-check decorators)
│   ├── inspections/
│   │   ├── routes.py
│   │   ├── models.py                (inspections_master, official_notices)
│   │   └── services.py
│   ├── rules_engine/
│   │   ├── engine.py                (deterministic rule evaluation)
│   │   ├── classifiers.py           (declaration field classification)
│   │   ├── config_models.py         (legal_rules_config)
│   │   └── tests/
│   ├── vision/
│   │   ├── preprocess.py            (OpenCV deskew/contrast/perspective)
│   │   ├── calibration.py           (px-to-mm calibration engine)
│   │   ├── ocr.py                   (Tesseract + Cloud Vision fallback)
│   │   └── barcode.py               (PyZbar/ZXing decoding)
│   ├── verification/
│   │   ├── gs1_gepir.py
│   │   ├── fssai_foscos.py
│   │   └── cache_models.py          (master_registry_cache)
│   ├── sentiment/
│   │   ├── scraper.py               (Playwright/Scrapy jobs)
│   │   └── roberta_scorer.py
│   ├── reporting/
│   │   ├── pdf_generator.py         (ReportLab/PDFKit)
│   │   └── hashing.py               (SHA-256 evidence hashing)
│   ├── mongo/
│   │   ├── ocr_dumps.py
│   │   ├── audit_reports.py
│   │   └── risk_scores.py
│   ├── batch/
│   │   ├── tasks.py                 (Celery tasks)
│   │   └── routes.py
│   ├── admin/
│   │   ├── routes.py                (rules-config editor, user mgmt, audit log)
│   └── dashboard/
│       └── routes.py                (KPI/heatmap aggregation endpoints)
├── migrations/                       (Alembic)
├── tests/
├── celery_worker.py
├── wsgi.py
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

---

## 3. Milestone Acceptance Criteria Summary

| Phase | Primary Deliverable | Acceptance Signal |
|---|---|---|
| 1 | Auth + DB skeleton | Role-based login and route-guarding works for all four roles |
| 2 | Capture + OCR pipeline | Raw OCR/barcode extraction returned for a real test image |
| 3 | Rule engine + calibration | Correct PASS/WARNING/FAIL verdicts on known-good and known-bad test images |
| 4 | External integrations + reporting | PDF report generated with verified hash; batch crawl produces triage-able results |
| 5 | Dashboards + hardened RBAC + deployment | End-to-end flows A, B, and C pass manual test on a deployed staging environment |
