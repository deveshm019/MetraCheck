# MetraCheck — UI/UX Design Brief

**Companion document to:** PRD v1.0, App Flow v1.0

---

## 1. Design Vision & Aesthetic Direction

MetraCheck is a **government-grade utility interface** used in the field under real-world conditions (bright sunlight, gloved hands, limited time per screen). The design vision is:

- **Professional and high-contrast** — favors clarity and legibility over decorative flourish, similar in tone to enforcement/regulatory tooling rather than a consumer lifestyle app.
- **Status-first visual language** — compliance status (Compliant / Warning / Non-Compliant) must be instantly legible at a glance, using consistent color + icon + label combinations everywhere it appears (scan results, history lists, dashboard cards).
- **Minimal cognitive load in the field** — the Inspector's capture-to-report flow uses large touch targets, few fields per screen, and progressive disclosure (detail is available on demand but not forced upfront).
- **Data-dense but organized for supervisory/admin views** — dashboards and tables can be denser, since these are typically used at a desk, not in the field.
- **Trustworthy, evidentiary feel** — typography and layout choices for reports and evidence views should read as credible, audit-ready documentation, not a casual app screen.

---

## 2. Color Palette

| Role | Color Name | Hex | Usage |
|---|---|---|---|
| Primary | DoCA Navy | `#1B3A5C` | Headers, primary buttons, navigation bar |
| Primary Dark | Deep Navy | `#102944` | Header background, active nav item |
| Secondary | Metrology Teal | `#0E7C7B` | Secondary actions, links, active tab indicators |
| Background | Neutral Canvas | `#F5F7FA` | App background |
| Card Surface | White | `#FFFFFF` | Cards, panels, modals |
| Border / Divider | Cool Gray | `#D8DEE6` | Borders, table dividers |
| Text Primary | Charcoal | `#1F2937` | Body text, headings |
| Text Secondary | Slate | `#5B6572` | Captions, helper text |
| **Success (Compliant)** | Compliance Green | `#1E8E3E` | Pass badges, compliant status |
| Success Background | Green Tint | `#E6F4EA` | Compliant card backgrounds |
| **Warning** | Amber | `#B76E00` | Warning badges (borderline results) |
| Warning Background | Amber Tint | `#FFF4E0` | Warning card backgrounds |
| **Danger (Violation)** | Enforcement Red | `#C5221F` | Fail badges, non-compliant status |
| Danger Background | Red Tint | `#FBE7E6` | Non-compliant card backgrounds |
| Info | Steel Blue | `#2E6DA4` | Informational banners, "unverified/unavailable" states |

**Note:** All status colors are paired with an icon and a text label (not color alone) to meet accessibility standards for color-blind users, per WCAG guidance.

---

## 3. Typography Scale

| Style | Font Family | Size | Weight | Usage |
|---|---|---|---|---|
| Display / H1 | Inter (or system sans-serif fallback) | 28px | 700 (Bold) | Page titles ("Inspection Results") |
| H2 | Inter | 22px | 600 (Semibold) | Section headers (dashboard cards, report sections) |
| H3 | Inter | 18px | 600 (Semibold) | Card titles, rule names |
| Body | Inter | 15px | 400 (Regular) | Standard body text |
| Body Emphasis | Inter | 15px | 600 (Semibold) | Field labels, key values (MRP, Net Qty) |
| Caption | Inter | 13px | 400 (Regular) | Timestamps, helper text, metadata |
| Badge / Status Label | Inter | 13px | 700 (Bold), uppercase, letter-spacing 0.02em | Compliant / Warning / Non-Compliant badges |
| Monospace | JetBrains Mono / system monospace | 14px | 400 | Raw extracted values, hashes, barcode numbers |

---

## 4. UI Component System

### 4.1 Buttons
- **Primary button:** Navy fill (`#1B3A5C`), white text, rounded corners (8px radius). Used for the single main action per screen (e.g., "Start Scan," "Generate Report").
- **Secondary button:** White fill, navy border and text. Used for alternate actions (e.g., "Retake Photo," "Add Note").
- **Destructive/Danger button:** Red border/text on white, reserved for actions like discarding a capture — never used for the "Non-Compliant" status itself (status uses badges, not buttons).
- Minimum touch target: 44×44px for all field-facing (Inspector) screens.

### 4.2 WebRTC Scanner Viewport
- Full-bleed live camera feed with a semi-transparent framing guide (rounded rectangle) indicating where to position the package panel.
- Optional secondary guide marker for calibration reference object placement, shown as a small corner indicator with a tooltip ("Place calibration card here for accurate font measurement").
- Capture button: large circular shutter button, bottom-center, thumb-reachable.
- Live quality indicator (small badge, e.g., "Good lighting" / "Too dark") updates in real time.
- Thumbnail strip of captured frames (front/back/barcode) along the bottom edge, tappable to review/retake.

### 4.3 Pass/Fail Compliance Badges
- Pill-shaped badge, icon + label, using the status color system from §2:
  - ✅ **COMPLIANT** — green fill/tint, checkmark icon.
  - ⚠️ **WARNING** — amber fill/tint, triangle-exclamation icon.
  - ❌ **NON-COMPLIANT** — red fill/tint, X-circle icon.
  - ℹ️ **UNVERIFIED** — steel-blue fill/tint, info icon (used for external-registry-unavailable states, distinct from the three statutory verdicts).
- Badge appears consistently in: scan result header, inspection list rows, dashboard summary tiles, and the PDF report header.

### 4.4 Rule Inspection Cards
- One card per evaluated rule (Rule 6 declarations, Rule 7 font height, Rule 8 net quantity, Rule 9 MRP).
- Card layout: rule name + short description (top), status badge (top-right), expandable detail section showing "Expected" vs. "Observed" values, and a thumbnail crop of the relevant image region with the detected bounding box overlaid.
- Cards are collapsible; failed/warning cards auto-expand by default, passed cards start collapsed to reduce visual noise.

### 4.5 Data Tables (Supervisor/Admin views)
- Sticky header row, zebra-striped rows for readability at density.
- Status column always uses the badge component (§4.3), never plain text.
- Sortable columns for date, status, region; filter bar above the table (date range picker, status multi-select, region dropdown, free-text search).
- Row click opens the case detail view; no destructive actions available directly from row-level controls.

### 4.6 Modal Dialogs
- Used for: manual calibration input, override justification entry, notice issuance confirmation.
- Centered, max-width 480px on desktop, full-screen sheet on mobile widths.
- Always include a clear title, primary action, and a secondary "Cancel" — no modal without an explicit dismiss path.

---

## 5. Layout Breakpoints & Responsive Guidelines

| Breakpoint | Width Range | Primary Consumers | Layout Notes |
|---|---|---|---|
| Mobile | < 640px | Field Inspector (in-field capture flow) | Single-column, bottom-anchored primary actions, full-bleed scanner viewport, collapsible sections |
| Tablet | 640–1024px | Field Inspector (office review), Auditor triage | Two-column layout for review screens (image left, extracted fields right) |
| Desktop | > 1024px | Supervisory Officer dashboard, System Admin console | Full multi-column dashboards, side navigation rail, data tables with all columns visible |

**General responsive rules:**
- The WebRTC scanner and capture-review flow are designed **mobile-first** since Field Inspectors are the primary users of that flow.
- Dashboard and analytics views are designed **desktop-first** with a simplified, KPI-summary-only fallback for narrower viewports (full tables/heatmaps are desktop-only for practicality).
- Navigation: bottom tab bar on mobile widths for the Inspector role; left side navigation rail on tablet/desktop for Supervisor/Admin roles.
