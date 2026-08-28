# MetraCheck — Backend Schema & Data Models

**Companion document to:** TRD v1.0

---

## 1. Architectural Separation: PostgreSQL vs. MongoDB

| Concern | Store | Rationale |
|---|---|---|
| User accounts, authentication, RBAC | PostgreSQL | Strong relational integrity needed for auth/permissions |
| Master government registries (cached GS1/FSSAI lookups) | PostgreSQL | Structured, relational, joined frequently against inspections |
| Rule threshold configuration | PostgreSQL | Small, structured, versioned configuration data |
| Structured audit logs (who did what, when) | PostgreSQL | Requires strong consistency and relational querying for compliance/legal purposes |
| Official notices | PostgreSQL | Structured records linked 1:1 with inspections |
| Raw/unparsed OCR output (variable-shape JSON, bounding boxes) | MongoDB | Schema varies by image/product; document model avoids rigid column definitions |
| Full compliance audit reports (nested rule verdicts, evidence refs) | MongoDB | Deeply nested, variable-length rule-evaluation structures fit a document model better than normalized tables |
| Product risk/sentiment scores | MongoDB | Semi-structured scraped data, evolving fields as sources change |
| Geospatial inspection heatmap source points | MongoDB (with geospatial indexing) | Flexible for aggregation pipelines feeding the heatmap |

**Rule of thumb applied throughout:** if a record must participate in strict relational integrity (foreign keys, RBAC, financial/legal accountability of *who* did *what*), it lives in PostgreSQL. If a record is a rich, variably-shaped *result* of an automated pipeline (OCR dump, nested rule report, scraped sentiment data), it lives in MongoDB and is linked back to its PostgreSQL parent record by ID reference.

---

## 2. PostgreSQL Schema

### 2.1 `users`

| Column | Type | Constraints |
|---|---|---|
| id | UUID | PRIMARY KEY, default gen_random_uuid() |
| email | VARCHAR(255) | UNIQUE, NOT NULL |
| password_hash | VARCHAR(255) | NOT NULL |
| full_name | VARCHAR(255) | NOT NULL |
| role | ENUM('INSPECTOR','SUPERVISOR','ADMIN','AUDITOR') | NOT NULL |
| jurisdiction_region | VARCHAR(255) | NULLABLE — state/district assignment |
| is_active | BOOLEAN | NOT NULL, DEFAULT true |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() |
| last_login_at | TIMESTAMPTZ | NULLABLE |

### 2.2 `inspections_master`

| Column | Type | Constraints |
|---|---|---|
| id | UUID | PRIMARY KEY |
| inspector_id | UUID | FOREIGN KEY → users(id), NOT NULL |
| source_type | ENUM('FIELD_CAPTURE','ECOMMERCE_BATCH') | NOT NULL |
| product_name_declared | VARCHAR(500) | NULLABLE — as extracted/declared |
| brand_name | VARCHAR(255) | NULLABLE |
| barcode_gtin | VARCHAR(20) | NULLABLE, INDEXED |
| location_lat | DOUBLE PRECISION | NULLABLE |
| location_lng | DOUBLE PRECISION | NULLABLE |
| region | VARCHAR(255) | NULLABLE, INDEXED |
| overall_status | ENUM('COMPLIANT','WARNING','NON_COMPLIANT','PROCESSING','FAILED') | NOT NULL, DEFAULT 'PROCESSING' |
| ocr_report_ref | VARCHAR(64) | MongoDB `compliance_audit_reports._id` reference |
| evidence_image_urls | TEXT[] | Array of S3/Cloudinary URLs |
| evidence_sha256_hashes | TEXT[] | Parallel array of hashes for each evidence image |
| notice_id | UUID | FOREIGN KEY → official_notices(id), NULLABLE |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() |

**Relationships:** `inspections_master.inspector_id → users.id` (many-to-one); `inspections_master.notice_id → official_notices.id` (one-to-one, nullable until a notice is issued).

### 2.3 `legal_rules_config`

| Column | Type | Constraints |
|---|---|---|
| id | UUID | PRIMARY KEY |
| rule_code | VARCHAR(20) | NOT NULL — e.g. 'RULE_7_FONT_HEIGHT' |
| description | TEXT | NOT NULL |
| net_qty_band_min | NUMERIC | NULLABLE — lower bound of applicable net-quantity band |
| net_qty_band_max | NUMERIC | NULLABLE — upper bound |
| net_qty_unit | VARCHAR(10) | NULLABLE — g / kg / ml / l |
| min_font_height_mm | NUMERIC(4,2) | NULLABLE — applies to Rule 7 rows |
| mrp_regex_pattern | TEXT | NULLABLE — applies to Rule 9 rows |
| accepted_units | TEXT[] | NULLABLE — applies to Rule 8 rows |
| is_active | BOOLEAN | NOT NULL, DEFAULT true |
| version | INTEGER | NOT NULL, DEFAULT 1 |
| effective_from | DATE | NOT NULL |
| updated_by | UUID | FOREIGN KEY → users(id) |
| updated_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() |

This table is the **single source of truth** the rule engine reads at evaluation time — it is never hardcoded in application logic, so an Admin can update thresholds (e.g., if the LMPC Rules are amended) without a code deployment.

### 2.4 `official_notices`

| Column | Type | Constraints |
|---|---|---|
| id | UUID | PRIMARY KEY |
| inspection_id | UUID | FOREIGN KEY → inspections_master(id), NOT NULL |
| issued_by | UUID | FOREIGN KEY → users(id), NOT NULL — must have role SUPERVISOR or ADMIN |
| notice_reference_number | VARCHAR(100) | UNIQUE, NOT NULL |
| notice_type | VARCHAR(100) | NOT NULL — e.g. 'SHOW_CAUSE', 'PENALTY' |
| status | ENUM('DRAFT','ISSUED','RESOLVED','APPEALED') | NOT NULL, DEFAULT 'DRAFT' |
| issued_at | TIMESTAMPTZ | NULLABLE |
| notes | TEXT | NULLABLE |
| created_at | TIMESTAMPTZ | NOT NULL, DEFAULT now() |

### 2.5 Supplementary reference tables (summary)

- `master_registry_cache` — cached GS1/FSSAI lookup results (gtin/license_number, resolved_entity_name, resolved_address, verification_status, last_checked_at) to avoid repeated external calls.
- `audit_log` — append-only: (id, actor_id, action, entity_type, entity_id, before_value JSONB, after_value JSONB, timestamp).

---

## 3. MongoDB Schema

### 3.1 `unparsed_ocr_dumps`

```json
{
  "_id": "ObjectId",
  "inspection_id": "UUID (references PostgreSQL inspections_master.id)",
  "image_ref": "string (S3/Cloudinary URL of the source image this OCR run was performed on)",
  "ocr_engine": "string ('tesseract' | 'google_cloud_vision')",
  "processed_at": "ISODate",
  "text_blocks": [
    {
      "block_id": "string",
      "raw_text": "string",
      "confidence": "float (0-1)",
      "bounding_box": {
        "x": "int", "y": "int", "width": "int", "height": "int"
      },
      "line_char_height_px_median": "float"
    }
  ],
  "barcode_results": [
    {
      "symbology": "string ('EAN13' | 'GS1_DATAMATRIX' | 'QR')",
      "decoded_value": "string",
      "bounding_box": { "x": "int", "y": "int", "width": "int", "height": "int" }
    }
  ],
  "calibration": {
    "method": "string ('reference_object' | 'package_relative' | 'manual')",
    "px_per_mm": "float",
    "confidence": "string ('high' | 'medium' | 'low')"
  }
}
```

### 3.2 `compliance_audit_reports`

```json
{
  "_id": "ObjectId",
  "inspection_id": "UUID (references PostgreSQL inspections_master.id)",
  "generated_at": "ISODate",
  "overall_status": "string ('COMPLIANT' | 'WARNING' | 'NON_COMPLIANT')",
  "extracted_declarations": {
    "manufacturer_name": "string | null",
    "manufacturer_address": "string | null",
    "net_quantity_value": "float | null",
    "net_quantity_unit": "string | null",
    "mrp_value": "float | null",
    "mrp_text_raw": "string | null",
    "mfg_or_pack_date": "string | null",
    "consumer_care_details": "string | null",
    "fssai_license_number": "string | null",
    "country_of_origin": "string | null"
  },
  "rule_verdicts": [
    {
      "rule_code": "string (e.g. 'RULE_7_FONT_HEIGHT')",
      "description": "string",
      "expected": "string",
      "observed": "string",
      "status": "string ('PASS' | 'WARNING' | 'FAIL')",
      "evidence_bounding_box_ref": "string (block_id in unparsed_ocr_dumps)",
      "measured_font_height_mm": "float | null"
    }
  ],
  "manual_overrides": [
    {
      "rule_code": "string",
      "overridden_by_user_id": "UUID",
      "justification": "string",
      "created_at": "ISODate"
    }
  ],
  "external_verification": {
    "gs1_gepir": {
      "status": "string ('verified' | 'mismatch' | 'unavailable')",
      "resolved_entity": "string | null"
    },
    "fssai_foscos": {
      "status": "string ('verified' | 'invalid' | 'unavailable')",
      "license_holder_name": "string | null"
    }
  },
  "evidence_image_urls": ["string"],
  "evidence_sha256_hashes": ["string"],
  "report_pdf_url": "string | null"
}
```

### 3.3 `product_risk_scores`

```json
{
  "_id": "ObjectId",
  "gtin": "string",
  "product_name": "string",
  "brand_name": "string",
  "sentiment_summary": {
    "sample_size_reviews": "int",
    "risk_keywords_detected": ["string (e.g. 'fake', 'expired', 'leaking')"],
    "sentiment_risk_score": "float (0-1, advisory only, NOT a compliance score)",
    "model_used": "string ('roberta-base-sentiment')"
  },
  "data_sources": ["string (e.g. 'amazon_in', 'flipkart')"],
  "last_scraped_at": "ISODate",
  "disclaimer": "This score is advisory consumer-sentiment context and is not a statutory Legal Metrology compliance determination."
}
```

---

## 4. Authentication & Authorization Flow

```
1. POST /auth/login {email, password}
       │
       ▼
   Verify password_hash (bcrypt/argon2) against users table
       │
       ▼
   Issue:
     - access_token  (JWT, short expiry ~15-30 min,
                       claims: user_id, role, jurisdiction_region)
     - refresh_token  (opaque or JWT, longer expiry, stored
                       hashed server-side for revocation support)
       │
       ▼
2. Client includes: Authorization: Bearer <access_token>
   on every subsequent API request
       │
       ▼
3. Flask middleware (per request):
     - Validate JWT signature & expiry
     - Extract role claim
     - Check role against the endpoint's required-role list
       (RBAC matrix, e.g. POST /admin/rules-config requires ADMIN)
     - Reject with 401/403 if invalid/insufficient
       │
       ▼
4. On access_token expiry:
     POST /auth/refresh {refresh_token}
       │
       ▼
   Validate refresh_token against stored hash + expiry;
   issue new access_token (+ rotate refresh_token)
       │
       ▼
5. Logout: refresh_token is revoked (deleted/blacklisted)
   server-side; client discards both tokens
```

### 4.1 Role Middleware Matrix (summary)

| Endpoint group | INSPECTOR | SUPERVISOR | ADMIN | AUDITOR |
|---|---|---|---|---|
| `/inspector/*` (capture, own history) | ✅ | ✅ (read) | ✅ (read) | ❌ |
| `/supervisor/*` (dashboard, notices) | ❌ | ✅ | ✅ | ❌ (read-only case view only) |
| `/auditor/*` (batch crawl) | ❌ | ✅ (read) | ✅ | ✅ |
| `/admin/*` (users, rules-config, audit log) | ❌ | ❌ | ✅ | ❌ |
| `/repository/search` | ✅ (own scope) | ✅ (jurisdiction scope) | ✅ (all) | ✅ (own scope) |
