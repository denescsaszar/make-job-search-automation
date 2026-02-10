# Make Scenario Mapping

## Overview

Maps the **Make.com scenario modules** to their purpose, configuration, and connections. This reflects the **currently deployed** scenario (ID: 4388094).

---

## Scenario: Job Ingestion Pipeline

**Trigger:** Scheduled daily at 09:05 (Europe/Berlin)
**Operations per run:** 21 (1 HTTP + 10 creates + 10 appends)

---

## Current Module Map (4 Modules)

```
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 4: HTTP Request — SerpAPI                          [TRIGGER] │
│ Type: HTTP > Make a request                                         │
│ Schedule: Daily at 09:05, Europe/Berlin                             │
│ Method: GET                                                         │
│ URL: https://serpapi.com/search.json                                │
│ Query params:                                                       │
│   engine = google_jobs                                              │
│   q = Technical Product Manager                                     │
│   location = New York, NY, United States                            │
│   gl = us                                                           │
│   hl = en                                                           │
│   api_key = (stored in module)                                      │
│ Output: 131 KB JSON with jobs_results[] (10 items)                  │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 8: Iterator — Jobs                                           │
│ Type: Flow Control > Iterator                                       │
│ Array: {{4.data.jobs_results}} (green pill, NOT literal text)       │
│ Output: 10 bundles, each with:                                      │
│   {{8.title}}, {{8.company_name}}, {{8.location}},                  │
│   {{8.share_link}}, {{8.description}}                               │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 10: Notion — Create Database Item (Legacy)                   │
│ Type: Notion > Create a Database Item (Legacy)                      │
│ Connection: "Notion – Jobs Master (Shared DB)"                      │
│ Database: job-agent                                                 │
│ Properties:                                                         │
│   Position (Title)  = {{8.title}}                                   │
│   Company           = {{8.company_name}}                            │
│   Place             = {{8.location}}                                │
│   Posting URL       = {{8.share_link}}                              │
│   Status            = "To-do: Found"                                │
│   Contact           = (empty)                                       │
│   Job Description   = (empty — written via Append instead)          │
│ Output: Database Item ID (used by Module 17)                        │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 17: Notion — Append Page Content                             │
│ Type: Notion > Append a Page Content                                │
│ Connection: "Notion – Jobs Master (Shared DB)"                      │
│ Page ID: {{10.id}} (Database Item ID from Module 10)                │
│ Content Objects:                                                    │
│   Item 1:                                                           │
│     Type: Paragraph                                                 │
│     Text > Item 1 > Type: Text                                      │
│     Text > Item 1 > Content: {{8.description}}                      │
│ Purpose: Write full job description into page body                  │
│          (bypasses 2000-char rich text property limit)               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Module Summary Table

| ID | Module                   | Type         | Purpose                              | Ops/run |
|----|--------------------------|--------------|--------------------------------------|---------|
| 4  | HTTP Request (SerpAPI)   | HTTP         | Fetch job listings                   | 1       |
| 8  | Iterator (jobs)          | Flow Control | Split array into individual bundles  | 0       |
| 10 | Notion Create            | Notion       | Write job metadata to database row   | 10      |
| 17 | Notion Append            | Notion       | Write full description to page body  | 10      |

**Total ops per run:** 21

---

## Connection Requirements

| Connection                        | Service | Used By      | Auth Type |
|----------------------------------|---------|--------------|-----------|
| (HTTP module, API key in params) | SerpAPI | Module 4     | API Key   |
| Notion – Jobs Master (Shared DB) | Notion  | Modules 10, 17 | OAuth  |

---

## Planned Changes

### Step 1.5 — Pagination
Restructure to: Repeater → HTTP → Iterator → Notion Create → Notion Append → Set Variable.
See [pagination.md](pagination.md) for full plan.

### Step 2 — Deduplication
Add Notion Search before Create to check if share_link already exists. Use Router to skip duplicates.

### Step 2 — Multi-Query
Add Set Variable (query list) + outer Iterator to loop through multiple search terms and locations.
