# MetraCheck — Product Requirements Document (PRD)

**Project:** MetraCheck — AI-Powered Legal Metrology Compliance Verification System
**SIH 2026 Problem Statement:** ID 26034 — Department of Consumer Affairs (DoCA), Ministry of Consumer Affairs, Food & Public Distribution
**Document Owner:** MetraCheck Team
**Version:** 1.0
**Status:** Draft — Master Blueprint for Development Team & AI Coding Agents

---

## 1. Executive Overview & Problem Statement Context

### 1.1 Background

Every packaged commodity sold in India — whether on a supermarket shelf or an e-commerce listing — is legally required under the **Legal Metrology Act, 2009** and the **Legal Metrology (Packaged Commodities) Rules, 2011 (LMPC Rules)** to carry a defined set of mandatory declarations: manufacturer/packer/importer identity and address, net quantity in standard units, Maximum Retail Price (MRP) inclusive of taxes, month and year of manufacture/packing/import, consumer care contact details, and (under Rule 7) declarations printed at a minimum font height relative to the package's net quantity.

Today, verification of these declarations is performed **manually** by Legal Metrology Officers who physically inspect packages, read fine print, and — where font-height compliance is in question — attempt to measure text with tools such as scales or calipers held against the packaging. This process is:

- **Slow** — a thorough inspection of a single product can take 10–20 minutes.
- **Subjective** — font-height and placement judgments vary between inspectors.
- **Non-scalable** — thousands of SKUs are introduced or repackaged every year across offline retail and e-commerce, while inspector headcount is fixed.
- **Poorly documented** — evidence of a violation is often just handwritten notes or an unstructured photograph, weakening enforceability.

### 1.2 Problem Statement (Manual vs. Automated Inspection)

| Dimension | Manual Inspection (Current State) | MetraCheck (Automated State) |
|---|---|---|
| Time per product | ~10–20 minutes | Target: under 1 minute per capture (design goal, to be validated in pilot) |
| Font-height measurement | Estimated visually or with physical tools | Calibrated pixel-to-mm computation from image |
| Declaration completeness check | Manual checklist, memory-dependent | Rule engine checks all Rule 6–9 fields systematically |
| Evidence trail | Photographs + paper notes | Timestamped image + hashed PDF + structured audit record |
| Coverage of e-commerce listings | Effectively unaudited at scale | Batch-crawlable and screenable |
| Reporting to DoCA | Manual compilation | Centralized dashboard, searchable repository |

### 1.3 Statutory Basis

MetraCheck's validation logic is anchored directly to the LMPC Rules, 2011:

- **Rule 6** — Declarations to be made on every package (identity, net quantity, MRP, dates, consumer care, country of origin where applicable).
- **Rule 7** — Manner of declaration: minimum font/character height requirements scaled to net quantity, plus placement and prominence requirements.
- **Rule 8** — Declaration of net quantity, permissible units, and rounding rules.
- **Rule 9** — Declaration of MRP, retail sale price format, and "inclusive of all taxes" requirement.

The system's rule engine is a **deterministic, explainable** implementation of these clauses — not a black-box model — so that every flagged violation can be traced back to a specific rule and threshold value for legal defensibility.

---

## 2. High-Level Business & Statutory Objectives

1. **Reduce enforcement turnaround time** for Legal Metrology inspections by automating declaration extraction and rule checking.
2. **Standardize compliance judgment** across inspectors and states by replacing subjective assessment with a deterministic rule engine and calibrated measurement.
3. **Extend enforcement reach to e-commerce** by enabling batch screening of product listing images, not just physical retail inspection.
4. **Produce legally admissible evidence** — every inspection generates a cryptographically hashed, timestamped record linking the source image, extracted text, rule evaluation, and final verdict.
5. **Give DoCA leadership visibility** into compliance trends via a central analytics dashboard (violation types, geographic hotspots, repeat offenders).
6. **Keep the system assistive, not authoritative** — MetraCheck supports inspector decision-making; it does not autonomously issue legal notices. A human inspector or supervisory officer remains the decision-maker of record.
7. **Build on an open, low-cost technology stack** so the solution is financially viable for state-level and central deployment without recurring licensing costs.

---

## 3. User Persona Profiles

### 3.1 Field Inspector (Legal Metrology Officer)
- **Context:** Visits retail stores, warehouses, and markets; performs on-the-spot inspections.
- **Goals:** Quickly capture a product image, get an instant compliance read-out, attach evidence, and generate a report without needing technical expertise.
- **Pain points today:** Carries measuring tools, fills paper forms, must manually recall every Rule 6–9 requirement.
- **Device context:** Primarily a smartphone or tablet browser in the field; may also use a desktop when back at the office.
- **Key needs:** Fast capture flow, clear pass/fail/warning indicators, offline-tolerant capture queue, minimal typing.

### 3.2 Supervisory Enforcement Officer
- **Context:** Oversees a team of field inspectors across a jurisdiction (district/state).
- **Goals:** Monitor inspection volume and violation trends, review flagged evidence before a notice is issued, approve/escalate cases.
- **Pain points today:** No consolidated view of inspection activity; violations tracked in disparate paper files.
- **Key needs:** Dashboard with filters (region, date, violation type, severity), drill-down into individual inspection evidence, ability to issue an official notice from a validated case.

### 3.3 System Administrator
- **Context:** IT/technical staff responsible for platform upkeep within DoCA or a state Legal Metrology department.
- **Goals:** Manage user accounts and roles, configure/update rule thresholds (e.g., Rule 7 font-height tables) as regulations are amended, maintain system health and audit integrity.
- **Key needs:** RBAC management console, rule configuration UI (not hardcoded values requiring redeployment), audit log access, system health visibility.

### 3.4 E-Commerce Compliance Auditor
- **Context:** A specialized reviewer (DoCA staff or delegated auditor) responsible for screening online marketplace listings for compliance.
- **Goals:** Run batch scans across product listing images from e-commerce platforms, triage flagged listings, and route confirmed violations into the same evidence/notice pipeline used for physical inspections.
- **Key needs:** Batch upload/crawl-ingestion interface, bulk triage view, ability to mark listings for follow-up by a supervisory officer.

---

## 4. Core Epics & Functional Requirements

Priority key: **P0** = must-have for MVP/demo, **P1** = required for a complete production system, **P2** = valuable enhancement, can follow later.

### Epic 1 — Image Capture & WebRTC Scanning

| ID | Requirement | Priority |
|---|---|---|
| E1-01 | Web app must access the device camera via the browser WebRTC/MediaDevices API for live capture (no native app install required). | P0 |
| E1-02 | Provide a framing overlay guiding the user to align the package panel and, where used, a reference calibration marker/card within frame. | P0 |
| E1-03 | Support capturing multiple images per product (front panel, back/ingredient panel, barcode close-up). | P0 |
| E1-04 | Support fallback image upload (from device gallery) when live camera capture is not usable. | P0 |
| E1-05 | Client-side basic quality check (blur/brightness heuristic) before accepting a capture, prompting retake if poor. | P1 |
| E1-06 | Queue captures locally when network is unavailable and sync automatically on reconnect. | P1 |
| E1-07 | Multi-frame capture assist (burst of 3 frames) to let the backend pick the sharpest for OCR. | P2 |

### Epic 2 — OCR, Spatial Bounding Box & Barcode Extraction

| ID | Requirement | Priority |
|---|---|---|
| E2-01 | Backend must preprocess uploaded images: deskew, perspective correction, contrast/adaptive threshold enhancement (OpenCV). | P0 |
| E2-02 | Run OCR (Tesseract) to extract all visible text with bounding box coordinates for each detected text block. | P0 |
| E2-03 | Decode 1D barcodes (EAN-13/GTIN-13) and 2D symbologies (GS1 DataMatrix/QR) present in the image (PyZbar/ZXing). | P0 |
| E2-04 | Classify extracted text blocks into declaration categories (MRP, net quantity, mfg/pack date, manufacturer address, consumer care, FSSAI number where present) using regex/keyword + positional heuristics. | P0 |
| E2-05 | Compute the physical font height in millimeters for each classified declaration block using the pixel-to-mm calibration ratio (see TRD §3). | P0 |
| E2-06 | Fall back to Google Cloud Vision API for low-confidence/degraded images where Tesseract confidence is below a configurable threshold. | P1 |
| E2-07 | Persist raw OCR JSON (all text blocks, confidences, coordinates) for audit and reprocessing purposes. | P0 |

### Epic 3 — Legal Metrology Rule Validation Engine

| ID | Requirement | Priority |
|---|---|---|
| E3-01 | Validate presence of all Rule 6 mandatory declarations; flag any missing declaration. | P0 |
| E3-02 | Validate MRP format against Rule 9 (numeric value present, "inclusive of all taxes" phrase present, correct currency symbol/format). | P0 |
| E3-03 | Validate Net Quantity against Rule 8 (standard unit used — g/kg/ml/l/number — and unit appropriate to product type). | P0 |
| E3-04 | Validate Mfd/Packed/Import date format and presence (month + year at minimum). | P0 |
| E3-05 | Validate manufacturer/packer/importer name and address completeness. | P0 |
| E3-06 | Validate consumer care detail presence (phone/email/address for grievance redressal). | P1 |
| E3-07 | Validate Rule 7 font height: compare the computed mm height of each mandatory declaration against the minimum threshold defined for the product's net-quantity band; flag under-sized text. | P0 |
| E3-08 | Rule threshold values (font-height bands, unit whitelist, regex patterns) must be stored in a configuration table, not hardcoded, so administrators can update them as rules are amended. | P0 |
| E3-09 | Output a structured, rule-by-rule verdict object (rule ID, expected value, observed value, pass/fail/warning) rather than a single opaque score. | P0 |
| E3-10 | Support a manual override workflow where an inspector can annotate a borderline/false-positive result with justification, without silently altering the underlying rule evaluation. | P1 |

### Epic 4 — External Registry Verification & Sentiment Scoring

| ID | Requirement | Priority |
|---|---|---|
| E4-01 | Cross-check decoded GTIN-13 barcode against GS1 GEPIR to resolve brand owner/company details where available. | P1 |
| E4-02 | Cross-check extracted FSSAI license number (for food products) against the FSSAI FoSCoS registry for validity. | P1 |
| E4-03 | Gracefully degrade (mark as "unverified — registry unavailable") when an external lookup times out or the registry has no public API, rather than blocking the inspection. | P0 |
| E4-04 | Cache registry lookup results locally to reduce repeated external calls for the same GTIN/FSSAI number. | P1 |
| E4-05 | Run NLP sentiment/risk analysis (RoBERTa-based) over available e-commerce customer reviews for a given product to produce a supplementary "consumer risk indicator" (not a compliance verdict). | P2 |
| E4-06 | Clearly label sentiment-derived scores as advisory context, separate from statutory rule verdicts, in both the UI and any generated report. | P0 |

### Epic 5 — Evidence Trail & Cryptographic PDF Report Generation

| ID | Requirement | Priority |
|---|---|---|
| E5-01 | Generate a PDF compliance report per inspection containing: source image(s), extracted declarations, rule-by-rule verdicts, overall status (Compliant/Warning/Non-Compliant), inspector identity, timestamp, and location (if available). | P0 |
| E5-02 | Compute a SHA-256 hash of the original captured image(s) at upload time and store it alongside the record to demonstrate the evidence has not been altered. | P0 |
| E5-03 | Store high-resolution evidence images in object storage (S3/Cloudinary) with access controlled by role. | P0 |
| E5-04 | Support export of the report in both PDF and an editable/structured format (e.g., JSON or DOCX) for downstream legal drafting. | P1 |
| E5-05 | Reports must be retrievable later by inspection ID, product, or date range without regenerating them from scratch. | P0 |

### Epic 6 — Enforcement Analytics Dashboard & Search Repository

| ID | Requirement | Priority |
|---|---|---|
| E6-01 | Dashboard showing inspection volume, violation rate, and violation type breakdown over a selectable date range. | P0 |
| E6-02 | Geographic heatmap of inspections/violations (where location data is captured). | P1 |
| E6-03 | Searchable repository of all past inspections filterable by product, brand, inspector, region, and compliance status. | P0 |
| E6-04 | Drill-down from a dashboard chart into the underlying list of inspection records. | P1 |
| E6-05 | Role-based dashboard views: Field Inspectors see their own inspection history; Supervisory Officers see their jurisdiction; Admins see system-wide data. | P0 |
| E6-06 | Ability for a Supervisory Officer to mark a case as "Notice Issued" and record the reference/date of the official notice. | P1 |

---

## 5. Non-Functional Requirements

### 5.1 Performance
- End-to-end processing (capture → extraction → rule verdict) should target under **10 seconds** per single-image inspection under normal network conditions (design target, to be validated in pilot testing — not a contractual SLA at MVP stage).
- Dashboard queries over the searchable repository should return within 2 seconds for typical filtered views (a jurisdiction/date range, not an unbounded full-table scan).
- Batch e-commerce crawling jobs run asynchronously (via a task queue) and must not block interactive inspector workflows.

### 5.2 Security
- All API traffic over HTTPS/TLS.
- JWT-based authentication with short-lived access tokens and refresh tokens.
- Role-Based Access Control enforced at the API layer for every endpoint (Inspector / Supervisor / Admin / Auditor).
- Evidence images and reports are immutable once generated; corrections require a new versioned record, not an in-place edit.
- All administrative actions (rule threshold changes, user role changes) are written to an append-only audit log.

### 5.3 Privacy
- No unnecessary personal data collection from consumers; captured images target product packaging, not identifiable bystanders.
- Inspector location data is used only for enforcement analytics and is access-restricted to supervisory/admin roles.

### 5.4 Offline Capabilities
- The field capture flow should tolerate intermittent connectivity: captures and metadata are queued locally (browser storage/IndexedDB) and synced when connectivity resumes.
- Full OCR/rule-engine processing requires backend connectivity and cannot run fully offline in the web-based MVP; this is a known scope boundary (a future edge/on-device model is a P2 consideration, out of scope for MVP).

### 5.5 Legal Admissibility
- Every generated report must be traceable to an unaltered, hashed source image and a deterministic, versioned rule-set — so the basis of any flagged violation can be reproduced and defended.
- The system explicitly frames outputs as **decision support**, not an automatic legal determination; final notices are issued by a human Supervisory Officer.

### 5.6 Scalability
- Architecture should support horizontal scaling of the OCR/vision processing tier independently of the web/API tier (e.g., via a task queue and worker pool) to absorb e-commerce batch-crawl spikes.

---

## 6. Out of Scope (MVP)

- Native mobile applications (iOS/Android) — MVP is a responsive Progressive Web App.
- Fully offline on-device OCR/rule evaluation.
- Automated legal notice issuance without human sign-off.
- Non-food/non-packaged-commodity category-specific rules beyond LMPC Rules, 2011 (e.g., BIS standards) — noted as a future extension point only.

---

## 7. Success Metrics (Design Targets, Pilot-Validated)

- Reduction in average inspection time per product versus manual baseline.
- Percentage of inspections with a complete, hash-verifiable evidence trail.
- Inspector adoption rate (active weekly users among onboarded inspectors).
- Number of e-commerce listings screened per week via batch audit.
- False-positive rate on Rule 7 font-height flags, as validated against manual re-checks during pilot.
