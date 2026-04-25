# RHR Inspection Report Management System — Design Document
**Version 1.0 | Draft for Review | Mar 21, 2026**

---

## 1. SYSTEM OVERVIEW

A unified inspection lifecycle management system that:
- Pulls project data from Zoho Books (time tracking)
- Generates state-compliant and custom inspection report PDFs
- Tracks deficiencies through to resolution
- Automates estimates → invoices → scheduling → completion reports
- Reminds of annual inspections on the correct anniversary

---

## 2. SUPPORTED INSPECTION TYPES

### NFPA 25 — Automatic Extinguishing Systems (AES)
State Fire Marshal mandated. Must be exact clones of official forms.
- **AES 2.1** — Annual Fire Sprinkler Inspection
- **AES 2.2** — 5-Year Fire Sprinkler Inspection
- **AES 9** — Fire Pump Test
- **AES 10** — Corrective Action Report (deficiency tracking)

### NFPA 72 — Fire Alarm
Custom version permitted (no State restrictions on format).
- Annual Inspection & Testing Report
- Supplemental: Notification Appliance
- Supplemental: Initiating Device

### Additional (future)
- Emergency Lighting
- Fire Extinguisher
- ERRCS

---

## 3. DATA ARCHITECTURE

### 3.1 Database: `inspections.db` (SQLite, hosted in container)

#### Table: `projects`
Links to Zoho Books. Single source of truth for property data.
```
id              INTEGER PRIMARY KEY
zoho_project_id TEXT UNIQUE         -- Zoho Books project ID
job_code        TEXT UNIQUE         -- e.g. 979AVE, 21660COP
property_name   TEXT                -- e.g. "979 Avenida Pico, San Clemente"
address         TEXT
city            TEXT
state           TEXT
zip             TEXT
customer_id     TEXT                -- Zoho Books customer ID
contact_name    TEXT
contact_phone   TEXT
contact_email   TEXT
created_at      DATETIME
last_synced     DATETIME            -- last Zoho Books API sync
```

#### Table: `inspections`
One record per inspection event.
```
id                  INTEGER PRIMARY KEY
project_id          INTEGER REFERENCES projects(id)
inspection_type     TEXT            -- "NFPA 72", "AES 2.1", etc.
inspection_date     DATE
inspector_name      TEXT
inspector_cert      TEXT            -- e.g. C10 RMQ
status              TEXT            -- pending | complete | deficiencies_open | deficiencies_resolved
report_pdf_path     TEXT            -- path to final generated PDF
report_data         JSON            -- all field values used to generate report (reused next year)
next_due_date       DATE            -- inspection_date + 1 year (or 5 years for AES 2.2)
zoho_estimate_id    TEXT            -- if deficiency estimate was created
zoho_invoice_id     TEXT            -- if invoice was generated
created_at          DATETIME
completed_at        DATETIME
```

#### Table: `deficiencies`
One record per deficiency found in an inspection.
```
id                  INTEGER PRIMARY KEY
inspection_id       INTEGER REFERENCES inspections(id)
description         TEXT            -- e.g. "Battery 12V 17.2AH dated 2/12/18 — replace"
location            TEXT            -- e.g. "FACP Panel Room"
severity            TEXT            -- info | minor | major | critical
status              TEXT            -- open | scheduled | repaired | verified
zoho_estimate_id    TEXT
zoho_invoice_id     TEXT
scheduled_date      DATE
resolved_date       DATE
resolution_notes    TEXT
created_at          DATETIME
```

#### Table: `report_templates`
Tracks which form version was used (important for SFM compliance).
```
id              INTEGER PRIMARY KEY
inspection_type TEXT
form_version    TEXT                -- e.g. "AES 2.1 Rev 2022"
template_path   TEXT
is_active       BOOLEAN
notes           TEXT
```

---

## 4. LIFECYCLE WORKFLOW

```
[1. PROJECT SYNC]
    │
    ▼
Zoho Books API → pull project name, address, contact, job_code
Store/update in projects table
    │
    ▼
[2. INSPECTION SCHEDULED]
    │
    ▼
Inspector completes field inspection
Data entered (via chat, web form, or jotform)
    │
    ▼
[3. REPORT GENERATION]
    │
    ├── No deficiencies → status: complete
    │       Report PDF generated (PASS)
    │       PDF saved to Project_Reports/<job_code>/
    │       PDF emailed to property contact
    │       Next inspection due date set (+1yr or +5yr)
    │       Annual reminder scheduled in Nanoclaw
    │
    └── Deficiencies found → status: deficiencies_open
            Deficiencies logged to deficiencies table
            AES 10 / Corrective Action form generated
            │
            ▼
        [4. ESTIMATE CREATION]
            Zoho Books API → create estimate for repairs
            Line items = one per deficiency
            Reference: "[JOB_CODE] — Repair deficiencies from [DATE] inspection"
            Estimate linked to project_id
            Estimate ID saved to inspection record
            │
            ▼
        [5. ESTIMATE APPROVED]  ← customer approves
            Zoho Books webhook or manual trigger
            Invoice auto-generated from estimate
            Invoice ID saved to inspection record
            │
            ▼
        [6. REPAIRS SCHEDULED]
            scheduled_date set on each deficiency
            Reminder set in Nanoclaw for scheduled date
            │
            ▼
        [7. REPAIRS COMPLETED]
            Each deficiency marked repaired + resolved_date
            │
            ▼
        [8. COMPLETION REPORT]
            New inspection record created (inspection_type = "AES 10 Completion")
            Updated AES 10 / NFPA 72 report generated showing PASS
            PDF emailed to property contact
            inspection status → deficiencies_resolved
            Next annual due date set
            Annual reminder scheduled
```

---

## 5. REPORT GENERATION ENGINE

### 5.1 Principles
- **Fully programmatic** — no AcroForm PDF templates (avoids fill anomalies)
- Python + ReportLab for server-side PDF generation
- Each report type has its own generator module
- All field values stored as JSON in `inspections.report_data` — reloaded next year as defaults

### 5.2 AES Forms — SFM Compliance
- Pixel-accurate clone of State Fire Marshal official forms
- Font: must match SFM spec (typically Helvetica/Arial)
- Layout dimensions verified against official form PDFs
- Form version tracked — if SFM updates forms, old inspections retain the version they were generated with
- **AES 10** doubles as the deficiency tracking form — generated at inspection time (open items) and again at completion (resolved)

### 5.3 NFPA 72 Custom Form
- RHR branding (logo, colors)
- All standard NFPA 72 sections maintained
- Supplemental pages (Notification Appliance, Initiating Device) auto-attached
- More flexibility on layout and design

### 5.4 Data Pre-fill Strategy
For each new inspection:
1. Pull project data from `projects` table (synced from Zoho)
2. Pull last inspection's `report_data` JSON for same type → use as defaults
3. Inspector confirms/updates values
4. Generate PDF
5. Save new `report_data` JSON to this inspection record

This means year 2 inspections start pre-filled with year 1's panel model, battery type, device counts, etc.

---

## 6. ZOHO BOOKS INTEGRATION

### 6.1 Project Sync (read)
- `GET /projects` → sync active projects to local DB
- Triggered: on demand, or daily scheduled task
- Stores: project name, customer, job code, contact

### 6.2 Estimate Creation (write)
- `POST /estimates` with:
  - `project_id` (top-level)
  - `reference_number`: `[JOB_CODE] — Deficiency Repairs [DATE]`
  - Line items from deficiencies table
- Estimate ID stored in `deficiencies` and `inspections`

### 6.3 Invoice Generation (write)
- `POST /invoices/fromestimate/{estimate_id}` after approval
- Invoice ID stored back in DB

### 6.4 Estimate/Invoice Status Polling
- Check estimate status periodically (approved/declined)
- Trigger invoice creation automatically on approval

---

## 7. SCHEDULING & REMINDERS

All reminders via Nanoclaw scheduled tasks:

| Event | Reminder |
|-------|----------|
| Inspection due (annual) | 30 days before + 7 days before |
| Scheduled repair date | Day before + day of |
| Estimate awaiting approval | 3 days after sending + weekly |
| Invoice unpaid | 30 days + 60 days after issue |

Annual reminders stored with:
- `job_code`
- `inspection_type`
- `due_date`
- Property contact email (auto-notify option)

---

## 8. NANOCLAW CHAT INTERFACE

Commands I can handle via chat:

```
#inspection create 979AVE NFPA72         → start new inspection
#inspection deficiency 979AVE "Battery expired"  → log deficiency
#inspection complete 979AVE              → mark inspection complete + generate report
#inspection status 979AVE               → show current status
#inspection history 979AVE              → list all inspections
#deficiency 979AVE list                 → list open deficiencies
#deficiency 979AVE resolve 3 "Replaced batteries 03-28-2026"
#estimate create 979AVE                 → create Zoho estimate for open deficiencies
#report email 979AVE grace@trendsystems.net  → email latest report PDF
#sync projects                          → pull latest from Zoho Books
```

---

## 9. FILE STORAGE

```
/workspace/group/reports/
    └── [JOB_CODE]/
        └── [INSPECTION_TYPE]/
            └── [JOB_CODE]_[TYPE]_[DATE].pdf
            └── [JOB_CODE]_[TYPE]_[DATE]_data.json

/workspace/group/inspections.db
```

---

## 10. DEVELOPMENT PHASES

### Phase 1 — Foundation
- [ ] Database schema creation (`inspections.db`)
- [ ] Zoho Books project sync module
- [ ] ReportLab generators for NFPA 72 + AES 2.1 (exact SFM clone)
- [ ] Basic chat commands: create, complete, generate report, email

### Phase 2 — Deficiency Tracking
- [ ] Deficiency logging and tracking
- [ ] AES 10 Corrective Action form generator
- [ ] Zoho estimate auto-creation from deficiencies

### Phase 3 — Full Lifecycle
- [ ] Estimate approval detection → auto invoice
- [ ] Repair scheduling + reminders
- [ ] Completion report generation
- [ ] Annual reminder scheduling

### Phase 4 — AES Forms (SFM Compliance)
- [ ] AES 2.1 exact clone (pending form review from Drive)
- [ ] AES 2.2 (5-year)
- [ ] AES 9 (Fire Pump)
- [ ] AES 10 (Corrective Action)

### Phase 5 — Polish
- [ ] Pre-fill from prior year data
- [ ] Multi-building projects
- [ ] Report history and search
- [ ] Customer portal link via Zoho estimate email

---

## 11. OPEN QUESTIONS FOR REVIEW

1. **AES form access** — need to review the SFM forms from the Google Drive link to confirm exact layout requirements for AES 2.1, 2.2, 9, 10
2. **Inspector data** — is it always Reice/RHR, or do we need multi-inspector support?
3. **Property contact notification** — should reports be auto-emailed to the building contact, or always sent manually?
4. **Estimate approval trigger** — manual (someone tells me "estimate approved") or automatic via Zoho webhook?
5. **rhrpdf relationship** — does this new system replace rhrpdf workflows, or run alongside it?
6. **Multi-building projects** — some properties (e.g. Chino Towne Center 12125 + 12217) have multiple buildings under one customer. How should inspections be grouped?

---

*Prepared by Claw · RHR Systems Inc. · Mar 21, 2026*
*Pending review before development begins.*
