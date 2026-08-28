# MetraCheck — App Flow & Navigation Map

**Companion document to:** PRD v1.0, TRD v1.0

---

## 1. Information Architecture Map

```
MetraCheck (Root)
│
├── /login                              (Public)
│
├── /inspector                          (Role: Field Inspector)
│   ├── /inspector/home                 (Recent inspections, quick-scan CTA)
│   ├── /inspector/scan                 (WebRTC scanner viewport)
│   ├── /inspector/review/:captureId    (OCR/extraction review & edit)
│   ├── /inspector/verdict/:captureId   (Rule engine results)
│   ├── /inspector/evidence/:captureId  (Attach notes/extra photos, confirm)
│   ├── /inspector/report/:inspectionId (Generated PDF preview/download)
│   └── /inspector/history              (My past inspections, searchable)
│
├── /supervisor                         (Role: Supervisory Officer)
│   ├── /supervisor/dashboard           (KPI summary, charts)
│   ├── /supervisor/heatmap             (Geographic violation heatmap)
│   ├── /supervisor/inspections         (Filterable inspection list)
│   ├── /supervisor/inspection/:id      (Full case detail + evidence)
│   └── /supervisor/notice/:id          (Issue/track official notice)
│
├── /auditor                            (Role: E-Commerce Compliance Auditor)
│   ├── /auditor/batches                (Batch crawl job list)
│   ├── /auditor/batch/new              (Configure a new batch scan)
│   ├── /auditor/batch/:id/results      (Triage flagged listings)
│   └── /auditor/listing/:id            (Individual listing detail, route to case)
│
├── /admin                              (Role: System Administrator)
│   ├── /admin/users                    (User & role management)
│   ├── /admin/rules-config             (Legal Metrology rule threshold editor)
│   ├── /admin/audit-log                (System audit trail)
│   └── /admin/system-health            (Job queue / integration status)
│
└── /repository                         (Shared, role-filtered)
    └── /repository/search              (Global searchable product/inspection repository)
```

---

## 2. User Flow Diagrams

### Flow A — Field Inspector: Capture to Report

```
[Login] 
   │
   ▼
[Inspector Home] ──(tap "New Scan")──▶ [WebRTC Scanner Viewport]
   │                                          │
   │                                          ▼
   │                              [Live camera + framing overlay
   │                               + optional calibration marker guide]
   │                                          │
   │                                (capture 1–3 images: front /
   │                                 back / barcode close-up)
   │                                          │
   │                                          ▼
   │                              [Client-side quality check]
   │                               (blur/brightness heuristic)
   │                                 │              │
   │                             good enough      too poor
   │                                 │              │
   │                                 ▼              └──▶ [Retake prompt]
   │                        [Upload to backend]
   │                                 │
   │                                 ▼
   │                  [Backend: Preprocess → OCR → Barcode
   │                   decode → Classify fields → Compute
   │                   font-height mm → Rule Engine evaluation]
   │                                 │
   │                                 ▼
   │                  [OCR / Extraction Review Screen]
   │               (inspector reviews auto-extracted fields,
   │                can correct obvious OCR misreads before
   │                final rule evaluation is locked)
   │                                 │
   │                                 ▼
   │                     [Rule Engine Validation Results]
   │              (rule-by-rule PASS/WARNING/FAIL cards;
   │               overall status: Compliant / Warning / Non-Compliant)
   │                                 │
   │                                 ▼
   │                  [Evidence Attachment Screen]
   │            (add inspector notes, optional extra photos,
   │             optional manual override with justification)
   │                                 │
   │                                 ▼
   │                  [Generate PDF Compliance Report]
   │             (SHA-256 hash computed & stored, PDF rendered)
   │                                 │
   │                                 ▼
   │                  [Report Preview / Download / Share]
   │                                 │
   ▼                                 ▼
[Inspector History] ◀───────── (inspection saved to repository)
```

### Flow B — Supervisory Officer: Dashboard to Notice

```
[Login]
   │
   ▼
[DoCA Dashboard]
 (KPIs: inspections this period, violation rate,
  top violation types)
   │
   ▼
[Geographic Heatmap] ──(optional detour)──▶ [drill into region]
   │
   ▼
[Inspection History — Filterable List]
 (filters: date range, region, inspector,
  compliance status, product/brand)
   │
   ▼
[Select a flagged case] ──▶ [Case Detail View]
                                  │
                     (review source images, extracted
                      declarations, rule verdicts,
                      SHA-256 hash confirmation,
                      inspector notes/override)
                                  │
                       ┌──────────┴───────────┐
                       ▼                       ▼
              [Approve → Issue           [Return to inspector
               Official Notice]           for re-inspection /
                       │                   more evidence]
                       ▼
        [Notice recorded: reference number,
         date, linked case — status updated
         to "Notice Issued"]
```

### Flow C — E-Commerce Batch Audit Flow

```
[Auditor Login]
   │
   ▼
[Batch Job List] ──(tap "New Batch")──▶ [Configure Batch Scan]
                                             (select platform/category/
                                              search terms or listing
                                              URL list; submit)
                                                     │
                                                     ▼
                                   [Job enqueued to Celery/Redis
                                    task queue — async crawl +
                                    per-listing image extraction
                                    + rule evaluation]
                                                     │
                                                     ▼
                                       [Batch Results — Triage View]
                                (listings grouped by status:
                                 Compliant / Warning / Non-Compliant /
                                 Could Not Process)
                                                     │
                                                     ▼
                                    [Select flagged listing] ──▶
                                    [Listing Detail — same evidence
                                     structure as a physical inspection]
                                                     │
                                                     ▼
                                    [Route to Supervisory Officer
                                     queue for notice consideration]
```

---

## 3. Edge Case & Error Handling Flows

### 3.1 Low Light / Blurry Capture

```
Capture taken
   │
   ▼
Client-side quality heuristic fails threshold
   │
   ▼
Show inline warning: "Image may be too blurry/dark for
accurate measurement — retake recommended"
   │
   ├─▶ [Inspector retakes] → proceed normally
   │
   └─▶ [Inspector proceeds anyway] → capture is tagged
        "low_quality_override": true → backend still
        processes but the report flags reduced confidence
        on any font-height verdicts derived from this image
```

### 3.2 Missing / Undecodable Barcode

```
Barcode decode step returns no result
   │
   ▼
Continue pipeline without barcode-dependent steps
(GS1 GEPIR lookup skipped, tagged "not available")
   │
   ▼
Rule engine still evaluates all label-based declarations
independently — a missing/unreadable barcode does NOT
block the Rule 6-9 declaration checks, since barcode
presence is not itself a mandatory declaration under
the LMPC Rules
   │
   ▼
Report notes: "Barcode not detected / undecodable —
GS1 verification unavailable for this inspection"
```

### 3.3 External API Timeout (GS1 GEPIR / FSSAI FoSCoS)

```
Registry lookup call issued with timeout (e.g., 5s)
   │
   ├─ Success → verification_status = "verified" / "mismatch"
   │
   └─ Timeout / error → verification_status = "unavailable"
        │
        ▼
   Inspection proceeds to completion; overall compliance
   status is driven ONLY by the deterministic rule engine
   (Rules 6-9), never blocked by an external verification
   failure. The report clearly labels registry checks as
   "unavailable" rather than silently omitting them.
```

### 3.4 Non-Compliant Font Size — Manual Override Path

```
Rule 7 engine verdict = FAIL (measured height below threshold)
   │
   ▼
Inspector reviews and believes measurement is inaccurate
(e.g., curved surface distortion suspected)
   │
   ▼
[Inspector opens "Add Override" on that specific rule card]
   │
   ▼
Inspector provides justification text + optionally a
manual two-point measurement recalibration
   │
   ▼
System stores override as a LINKED, separate record —
original automated FAIL verdict is preserved unmodified
   │
   ▼
Report displays BOTH: automated verdict AND inspector
override with justification, clearly distinguished
   │
   ▼
Case routed to Supervisory Officer with override flagged
for their explicit review before any notice is issued
```

### 3.5 Network Loss During Field Capture

```
Capture completed, upload attempted
   │
   ▼
Network unavailable
   │
   ▼
Capture + metadata queued in local browser storage
(IndexedDB), inspector sees "Queued — will sync
when back online" status badge
   │
   ▼
Connectivity restored (detected via online event /
periodic ping)
   │
   ▼
Queued items auto-upload in order; inspector notified
of successful sync per item
```
