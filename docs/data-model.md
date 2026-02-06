# Data Model & Field-Level Data Flow

## Overview

This document traces every field from its **raw source** through **transformation** to its **final Notion column**, showing exactly where each value comes from, how it's processed, and who owns it.

---

## Field-Level Data Flow Diagram

```
╔══════════════════════════════════════════════════════════════════════════════╗
║  SOURCE (SerpAPI Response)                                                  ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  jobs_results[n]                                                             ║
║  ├── title ─────────────────────┐                                           ║
║  ├── company_name ──────────────┤                                           ║
║  ├── location ──────────────────┤                                           ║
║  ├── description ───────────────┤  (future: used for AI scoring)            ║
║  ├── detected_extensions        │                                           ║
║  │   ├── posted_at ─────────────┤  (future: freshness filter)               ║
║  │   └── schedule_type ─────────┤  (future: full-time/part-time filter)     ║
║  └── related_links[]            │                                           ║
║      └── [0].link ──────────────┤                                           ║
║                                  │                                           ║
╚══════════════════════════════════╪═══════════════════════════════════════════╝
                                   │
                                   ▼
╔══════════════════════════════════════════════════════════════════════════════╗
║  TRANSFORMATION (Make Modules)                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  Iterator (splits jobs_results into individual bundles)                      ║
║  │                                                                          ║
║  ├── title ──────────────── pass-through ──────────────→ Position           ║
║  ├── company_name ─────── pass-through ──────────────→ Company             ║
║  ├── location ──────────── pass-through ──────────────→ Place              ║
║  ├── related_links[0].link                                                  ║
║  │   └── Filter: starts with "https://"                                    ║
║  │       ├── PASS ──────── pass-through ──────────────→ Posting URL        ║
║  │       └── FAIL ──────── DROP entire job (no write)                      ║
║  ├── (none) ────────────── static "Found" ────────────→ Status             ║
║  └── (none) ────────────── Notion auto-generated ─────→ Created At         ║
║                                                                              ║
║  Dedup Check (Notion Search on Posting URL)                                 ║
║  │                                                                          ║
║  ├── 0 results ──────────→ proceed to Create Page                          ║
║  └── 1+ results ─────────→ SKIP (no write)                                 ║
║                                                                              ║
║  Fit Score Calculation (Step 3)                                             ║
║  │                                                                          ║
║  ├── title ──────────── keyword match ────────────────→ Fit Score (0-100)  ║
║  ├── location ───────── whitelist match ──────────────→ (contributes)      ║
║  └── company_name ───── blacklist check ──────────────→ (contributes)      ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
                                   │
                                   ▼
╔══════════════════════════════════════════════════════════════════════════════╗
║  DESTINATION (Notion Database)                                              ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  ┌─────────────┬────────────┬──────────┬───────────┬────────────────────┐   ║
║  │ Field       │ Type       │ Owner    │ Required  │ Source Path        │   ║
║  ├─────────────┼────────────┼──────────┼───────────┼────────────────────┤   ║
║  │ Position    │ Title      │ Step 1   │ Yes       │ title              │   ║
║  │ Company     │ Rich text  │ Step 1   │ Yes       │ company_name       │   ║
║  │ Place       │ Rich text  │ Step 1   │ No        │ location           │   ║
║  │ Posting URL │ URL        │ Step 1   │ Yes*      │ related_links[0]   │   ║
║  │ Status      │ Select     │ Step 1   │ Yes       │ static: "Found"    │   ║
║  │ Created At  │ Date       │ Notion   │ Auto      │ auto-generated     │   ║
║  │ Fit Score   │ Number     │ Step 3   │ No        │ calculated         │   ║
║  │ Score Tier  │ Formula    │ Notion   │ Auto      │ derived from score │   ║
║  │ Applied     │ Checkbox   │ Step 5   │ No        │ agent-managed      │   ║
║  │ Notes       │ Rich text  │ Manual   │ No        │ user-entered       │   ║
║  └─────────────┴────────────┴──────────┴───────────┴────────────────────┘   ║
║                                                                              ║
║  * Posting URL is the deduplication key and filter gate.                    ║
║    Jobs without valid https:// URLs are never written.                      ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

## Field Ownership Rules

| Owner   | May Write           | May Read |
|---------|---------------------|----------|
| Step 1  | Position, Company, Place, Posting URL, Status | All |
| Step 3  | Fit Score           | All      |
| Notion  | Created At, Score Tier (formula) | All |
| Step 5  | Applied, Application Status | All |
| Manual  | Notes, tags         | All      |

**Rule:** Each field has exactly one write-owner. No two steps write to the same field. This prevents race conditions and makes debugging straightforward.

---

## SerpAPI Response Shape (Reference)

```json
{
  "jobs_results": [
    {
      "title": "Automation Consultant",
      "company_name": "Deloitte",
      "location": "Zurich, Switzerland",
      "description": "We are looking for...",
      "related_links": [
        {
          "link": "https://careers.deloitte.com/job/12345",
          "text": "Apply on Deloitte Careers"
        }
      ],
      "detected_extensions": {
        "posted_at": "2 days ago",
        "schedule_type": "Full-time"
      }
    }
  ]
}
```

---

## Future Fields (Planned)

| Field              | Type     | Owner  | Step | Purpose                       |
|--------------------|----------|--------|------|-------------------------------|
| Description        | Rich text| Step 1 | 2+   | For AI scoring context        |
| Posted At          | Date     | Step 1 | 2+   | Freshness filtering           |
| Schedule Type      | Select   | Step 1 | 2+   | Full-time / Part-time filter  |
| Source Query        | Rich text| Step 1 | 2+   | Which search query found this |
| Application Status | Select   | Step 5 | 5    | Applied / Interviewed / etc.  |
| Follow-Up Date     | Date     | Step 5 | 5    | Next action reminder          |
