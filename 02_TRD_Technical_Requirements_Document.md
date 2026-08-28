# MetraCheck — Technical Requirements Document (TRD)

**Companion document to:** PRD v1.0
**Scope:** Complete technical architecture, computer-vision pipeline design, rule engine specification, integration strategy, and security protocols.

---

## 1. Complete Architecture Blueprint

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER (Browser)                              │
│  React.js PWA  |  WebRTC Camera Capture  |  Tailwind CSS  |  Recharts       │
└───────────────────────────────┬───────────────────────────────────────────┘
                                 │ HTTPS / REST (JSON) + JWT
┌───────────────────────────────▼───────────────────────────────────────────┐
│                     API GATEWAY / APPLICATION LAYER                        │
│         Flask REST API  (Gunicorn WSGI, behind Nginx reverse proxy)        │
│   Auth & RBAC Middleware | Request Validation | Rate Limiting              │
└───────┬───────────────────────┬───────────────────────────┬───────────────┘
        │                       │                            │
        ▼                       ▼                            ▼
┌───────────────┐     ┌─────────────────────┐     ┌───────────────────────┐
│  SYNCHRONOUS   │     │   ASYNC TASK QUEUE   │     │   EXTERNAL SERVICES   │
│  PROCESSING    │     │   (Celery + Redis)    │     │   INTEGRATION LAYER   │
│  (single-image │     │  - E-commerce batch  │     │  - GS1 GEPIR          │
│   inspections) │     │    crawling          │     │  - FSSAI FoSCoS       │
│                │     │  - Bulk re-processing│     │  - Sentiment scraper  │
└───────┬────────┘     └──────────┬───────────┘     └───────────┬───────────┘
        │                          │                              │
        ▼                          ▼                              │
┌─────────────────────────────────────────────┐                   │
│         COMPUTER VISION & OCR PIPELINE        │                   │
│  OpenCV (deskew, perspective fix, contrast)   │                   │
│  Tesseract OCR / Google Cloud Vision (fallback)│                  │
│  PyZbar / ZXing (barcode decode)              │                   │
│  Spatial Calibration Engine (px → mm)         │                   │
└───────────────────────┬───────────────────────┘                   │
                         ▼                                          │
┌─────────────────────────────────────────────┐                     │
│       LEGAL METROLOGY RULE ENGINE             │                     │
│  Deterministic Python module evaluating       │                     │
│  Rules 6, 7, 8, 9 against extracted fields    │                     │
└───────────────────────┬───────────────────────┘◄────────────────────┘
                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         HYBRID DATA LAYER                            │
│  PostgreSQL (relational: users, RBAC, master registries, audit logs) │
│  MongoDB (documents: raw OCR JSON, bounding boxes, audit reports)    │
│  S3 / Cloudinary (evidence images, SHA-256 hashed)                   │
└─────────────────────────────────────────────┬───────────────────────┘
                                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  PDF EVIDENCE REPORT ENGINE (ReportLab/PDFKit)       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Detailed Tech Stack Matrix

| Layer | Technology | Purpose |
|---|---|---|
| Frontend | React.js (SPA/PWA) | Inspector & supervisor UI |
| Frontend | WebRTC / MediaDevices API | Live camera capture |
| Frontend | Tailwind CSS | Styling system |
| Frontend | Recharts / Chart.js | Dashboard visualizations |
| Backend API | Python Flask | REST API framework |
| Backend Server | Gunicorn (WSGI) + Nginx | Production application serving |
| Async Processing | Celery + Redis (broker) | Batch e-commerce crawling, bulk jobs |
| Computer Vision | OpenCV | Deskewing, contrast adjustment, perspective correction, spatial measurement |
| OCR | Tesseract OCR (pytesseract) | Primary text extraction |
| OCR (fallback) | Google Cloud Vision API | High-noise/curved-surface fallback |
| Barcode | PyZbar / ZXing | EAN-13 / GS1 DataMatrix decoding |
| Relational DB | PostgreSQL | Auth, RBAC, master registries, structured audit logs |
| Document DB | MongoDB | Raw OCR JSON, bounding boxes, nested compliance reports, geospatial heatmap source data |
| Object Storage | AWS S3 / Cloudinary | High-resolution evidence images |
| Integrity | SHA-256 hashing (hashlib) | Evidence tamper-detection |
| PDF Generation | ReportLab / PDFKit | Compliance report generation |
| Sentiment/NLP | RoBERTa (HuggingFace Transformers) | Consumer review risk scoring (advisory only) |
| Auth | JWT (PyJWT / Flask-JWT-Extended) | Stateless session auth |
| Deployment | Docker, Nginx | Containerized deployment |

---

## 3. Computer Vision & Spatial Calibration Blueprint

### 3.1 Objective

Rule 7 of the LMPC Rules mandates a **minimum font height** for mandatory declarations, scaled to the net quantity of the package. A camera image alone gives font height in **pixels**, which is meaningless without knowing the real-world scale of the shot. The calibration engine converts pixel measurements into millimeters.

### 3.2 Calibration Approaches (in order of preference)

1. **Reference-object calibration (primary approach):** The inspector places a known-size reference marker (e.g., a printed calibration card of fixed dimensions, or a standard-size coin/ruler edge) within the camera frame alongside the package. OpenCV detects the reference object's contour/ArUco marker and computes:

   ```
   px_per_mm = reference_object_pixel_width / reference_object_known_width_mm
   ```

2. **Package-relative calibration (secondary/fallback):** When many product categories have standardized panel dimensions (e.g., a known SKU database entry with declared package dimensions), the system can use the detected package panel edge length in pixels against the known physical panel dimension as an approximate calibration reference. This is lower-confidence and is flagged as "approximate calibration" in the report.

3. **Manual calibration override:** The inspector can manually input a known measurement (e.g., "this edge is 80mm") by tapping two points on the captured image in the UI; the system computes `px_per_mm` from that input. Always available as a fallback when no physical reference object is available.

### 3.3 Font Height Measurement Algorithm (High-Level)

```
1. Preprocess image: deskew (Hough line transform), perspective-correct
   (four-point transform if package edges are detected), enhance contrast
   (CLAHE / adaptive threshold).
2. Establish px_per_mm using calibration method (§3.2).
3. Run OCR to obtain bounding boxes for each text line/block.
4. Classify each text block into a declaration category
   (MRP / Net Qty / Mfg Date / Address / Consumer Care / Other) via
   regex + keyword + positional heuristics.
5. For each classified mandatory-declaration block:
     a. Compute bounding box height in pixels (cap-height estimate,
        using the block's median character height rather than full
        box height to reduce ascender/descender skew).
     b. Convert to millimeters: height_mm = height_px / px_per_mm
     c. Compare height_mm against the Rule 7 minimum threshold
        for the product's net-quantity band (from legal_rules_config).
     d. Record PASS / FAIL / WARNING (WARNING when within a small
        tolerance band of the threshold, to account for calibration
        and OCR bounding-box imprecision).
6. Persist all measurements (raw px, calibration ratio used,
   computed mm, threshold, verdict) to MongoDB for full traceability.
```

### 3.4 Known Limitations & Mitigations

| Limitation | Mitigation |
|---|---|
| Curved surfaces (bottles/cans) distort text and reference object apparent size | Multi-frame capture at different angles; perspective/cylindrical unwarping in OpenCV preprocessing |
| Low light or glare reducing OCR/measurement confidence | Client-side capture quality check before upload; server-side CLAHE contrast enhancement |
| No reference object available in frame | Manual two-point calibration fallback in UI |
| OCR bounding box includes excess whitespace/padding | Use median stroke height of detected characters rather than raw bounding box height where the OCR engine provides character-level boxes |

---

## 4. Rule Engine Specification

### 4.1 Design Principles

- **Deterministic:** Given the same extracted field values and the same configured thresholds, the engine always produces the same verdict. No generative model is used to decide compliance — the rule engine consumes AI-model *outputs* (OCR text, computed measurements) but the *decision logic* itself is explainable, versioned Python code plus data-driven configuration.
- **Explainable:** Every verdict object includes the rule ID, the expected condition, the observed value, and the pass/fail/warning outcome.
- **Configurable, not hardcoded:** Threshold values (font-height bands by net-quantity range, accepted unit strings, MRP regex pattern) live in the `legal_rules_config` table (see Backend Schema doc) so they can be updated by an Administrator when the LMPC Rules are amended, without a code deployment.

### 4.2 Rule Coverage

| Rule | What It Checks | Engine Logic Summary |
|---|---|---|
| Rule 6 — Mandatory declarations | Presence of: manufacturer/packer/importer name & address, net quantity, MRP, mfg/pack/import date, consumer care details, country of origin (where applicable) | For each required field, check that a classified text block of that category exists in the extraction output. Missing → FAIL. |
| Rule 7 — Font height & manner of declaration | Minimum character height for mandatory declarations, scaled to net-quantity band; placement/prominence | Computed mm height (§3.3) compared against threshold table keyed by net-quantity band. |
| Rule 8 — Net quantity declaration | Standard unit usage (g, kg, ml, l, or count), correct rounding/format | Regex-extract the numeric value + unit token; validate unit against the whitelist appropriate to the declared product type (weight vs. volume vs. count). |
| Rule 9 — MRP declaration | Presence of "Rs." / "₹" symbol, numeric value, "inclusive of all taxes" phrase | Regex pattern match against the classified MRP text block; flag if the mandatory tax-inclusive phrase is absent. |

### 4.3 Rule Evaluation Flow

```
extracted_fields (from OCR + classification)
        │
        ▼
for each active rule in legal_rules_config:
        │
        ├─ locate relevant extracted field(s)
        ├─ if field missing → verdict = FAIL ("declaration missing")
        ├─ if field present → apply rule-specific check function
        │        ├─ PASS  (within threshold)
        │        ├─ WARNING (borderline / low OCR confidence)
        │        └─ FAIL   (outside threshold / malformed)
        └─ append RuleVerdict{rule_id, expected, observed, status, evidence_ref}
        │
        ▼
aggregate verdicts → overall_status =
   FAIL if any rule = FAIL
   WARNING if no FAIL but at least one WARNING
   COMPLIANT if all PASS
```

### 4.4 Manual Override Handling

An inspector may attach an override annotation to a specific rule verdict (e.g., "font measured manually as compliant despite calibration uncertainty"). The override is stored as a **separate, linked record** — the original automated verdict is never overwritten — preserving a complete audit trail of both the automated assessment and the human judgment applied to it.

---

## 5. Third-Party Integration Strategy

| Integration | Purpose | Access Method | Notes |
|---|---|---|---|
| GS1 GEPIR | Resolve GTIN-13 barcode to brand owner / company registration | Public web portal (rate-limited) or paid GS1 India API | Cache resolved results; treat unavailability as "unverified," never as a compliance failure |
| FSSAI FoSCoS | Validate 14-digit FSSAI license numbers found on food labels | Public portal lookup (scrape) or third-party RegTech API | Same graceful-degradation principle as above |
| E-commerce review sources | Supply text for sentiment/risk scoring | Platform-provided data feeds where available, or compliant scraping via Playwright/Scrapy respecting each platform's terms of service and robots.txt | Output is advisory context only, never a compliance determination |

**Resilience pattern for all external integrations:** every call goes through a timeout + circuit breaker; on failure, the system records the field as `"verification_status": "unavailable"` and proceeds — external lookups must never block or fail the core inspection workflow.

---

## 6. Security & Integrity Protocols

### 6.1 Authentication & Authorization
- JWT access tokens (short expiry, e.g., 15–30 min) + refresh tokens (longer expiry, stored securely, rotated on use).
- Role claim embedded in the JWT; validated by Flask middleware on every protected route against the RBAC matrix (Inspector / Supervisor / Admin / E-Commerce Auditor).

### 6.2 Evidence Integrity
- SHA-256 hash computed on the original uploaded image bytes at ingestion, before any preprocessing, and stored immutably alongside the record.
- Any reprocessing (e.g., re-running OCR with an updated model) creates a new derived record referencing the original hashed image — the original is never overwritten.

### 6.3 Transport & Storage Security
- TLS enforced for all client-server and server-external-service communication.
- Object storage (S3/Cloudinary) buckets use signed, time-limited URLs for evidence image access rather than public URLs.
- Database credentials and API keys managed via environment variables / secrets manager, never committed to source control.

### 6.4 Audit Logging
- All authentication events, rule-config changes, and role changes written to an append-only PostgreSQL audit table with actor, timestamp, and before/after values.

---

## 7. Non-Functional Technical Considerations

- **Statelessness:** The Flask API tier is stateless (session state lives in JWT + database), allowing horizontal scaling behind a load balancer.
- **Idempotency:** Image upload and processing endpoints are designed to be safely retryable (client generates a request UUID; duplicate submissions with the same UUID are deduplicated server-side).
- **Observability:** Structured logging (JSON logs) from the Flask API and Celery workers, correlated by a request/trace ID that flows through to the stored inspection record.
