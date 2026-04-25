# Inspection Pipeline — Design Concept

**RHR Systems Inc.**  
**Status:** Design / Pre-development  
**Last updated:** 2026-04-25

---

## Concept

A lightweight inspection lifecycle tracker built on top of Zoho Projects. Each property inspection gets a dedicated task that serves as both a **live status board** and a **record of completion** — all data living in the project where it belongs.

The native Zoho task due date triggers the initial reminder. The agent (NanoClaw/Claudius) updates the pipeline status as each step is logged.

---

## Pipeline Stages

```
📅  Inspection Due        [DATE]
─────────────────────────────────────────
🔔  Notice sent to customer          ⬜
📋  Agreement sent & approved        ⬜
─────────────────────────────────────────
🔍  Inspection completed             ⬜
⚠️   Deficiencies found              —
─────────────────────────────────────────
📄  Report generated                 ⬜
📧  Report sent to customer          ⬜
─────────────────────────────────────────
🔧  Repair estimate created          ⬜
✔️   Repair estimate approved        ⬜
🏁  Repairs complete + final report  ⬜
```

Each step gets a date stamp when completed. The task description is updated via API on each state change.

---

## Data Flow

```
Zoho Projects Task
│
├── Description (splash page)
│     └── Live pipeline status board (emoji + dates)
│
├── Due Date
│     └── Native Zoho reminder → X days before inspection
│
├── Comments
│     └── Timestamped audit log of each action
│
└── Documents tab (WorkDrive)
      └── Inspection reports, estimates, deficiency photos
```

---

## Task Naming Convention

```
[YEAR] [TYPE] Inspection — [ProjectCode]
```

Examples:
- `2026 Annual Fire Alarm Inspection — 11667GOR`
- `2026 Sprinkler Inspection — 3380BLA`
- `2026 Backflow Test — 4100BLA`

---

## Inspection Types (proposed)

| Type | Typical Frequency | Authority |
|---|---|---|
| Fire Alarm (NFPA 72) | Annual | Local fire dept |
| Sprinkler (NFPA 25) | Annual + 5-yr | Local fire dept |
| Backflow Preventer | Annual | Water district |
| Exit / Emergency Lights | Annual | Building dept |
| Kitchen Hood Suppression | Semi-annual | Local fire dept |

---

## Agent Integration

NanoClaw/Claudius updates pipeline state via natural language:

> "Mark notice sent for 11667GOR inspection"
> "Inspection complete, deficiencies found — 11667GOR"
> "Report sent to customer — 3380BLA"

Each command:
1. Updates the task description status board
2. Posts a timestamped comment to the task
3. Optionally triggers next-step reminder or creates follow-on task

---

## Open Design Questions

- [ ] One task per inspection type, or combined annual task per property?
- [ ] Customer-facing portal link for report delivery confirmation?
- [ ] Auto-create repair estimate task when deficiencies = yes?
- [ ] Portfolio dashboard view across all projects?
- [ ] Integration with existing report generation workflow (rhrpdf)?

---

## Related

- [rhr-workflows](https://github.com/nottsatoshi/rhr-workflows) — operational workflow docs
- `rhr-workflows/zoho/Record_Project_Data_Workflow.md` — project data standards

---

> **Design concept — not yet implemented.**
> Returning to build after foundational Zoho Projects + WorkDrive integration is stable.
