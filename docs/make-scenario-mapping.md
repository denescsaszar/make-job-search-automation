# Make Scenario Mapping

## Overview

This document maps the **Make.com scenario modules** to their purpose, configuration, and connections. Use this as a reference when building or modifying the scenario in Make's visual editor.

---

## Scenario: Job Ingestion Pipeline

**Trigger:** Scheduled (every 6 hours)
**Operations per run:** ~3–5 per job (varies with dedup hits)

---

## Module Map

```
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 1: Scheduler                                                  │
│ Type: Trigger                                                        │
│ Config: Every 6 hours                                                │
│ Output: triggers execution                                           │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 2: Set Variable — Query List                                  │
│ Type: Tools > Set Variable                                           │
│ Config: variable = JSON array of search queries                      │
│ Output: {{query_list}} (array)                                       │
│                                                                      │
│ Value:                                                               │
│ [                                                                    │
│   {"q": "automation consultant", "location": "Switzerland"},         │
│   {"q": "API integration specialist", "location": "Zurich"},         │
│   {"q": "workflow automation", "location": "Remote"}                 │
│ ]                                                                    │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 3: Iterator — Queries                                         │
│ Type: Flow Control > Iterator                                        │
│ Input: {{query_list}}                                                │
│ Output: {{item.q}}, {{item.location}} (one per bundle)               │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 4: HTTP Request — SerpAPI                                     │
│ Type: HTTP > Make a request                                          │
│ Method: GET                                                          │
│ URL: https://serpapi.com/search.json                                 │
│ Query params:                                                        │
│   engine = google_jobs                                               │
│   q = {{item.q}}                                                     │
│   location = {{item.location}}                                       │
│   api_key = (from connection)                                        │
│ Output: {{data.jobs_results}} (array)                                │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 5: Iterator — Jobs                                            │
│ Type: Flow Control > Iterator                                        │
│ Input: {{data.jobs_results}}                                         │
│ Output: one job per bundle                                           │
│   {{item.title}}                                                     │
│   {{item.company_name}}                                              │
│   {{item.location}}                                                  │
│   {{item.related_links[1].link}}                                     │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 6: Filter — Valid URL                                         │
│ Type: Flow Control > Filter                                          │
│ Condition: {{item.related_links[1].link}} starts with "https://"     │
│ Label: "Has valid Posting URL"                                       │
│ Pass: continue to Module 7                                           │
│ Fail: bundle dropped silently                                        │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 7: Notion — Search (Dedup Check)                              │
│ Type: Notion > Search Objects                                        │
│ Database: Job Listings                                               │
│ Filter:                                                              │
│   Property: Posting URL                                              │
│   URL > equals > {{item.related_links[1].link}}                      │
│ Output: array of matching pages                                      │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 8: Router                                                     │
│ Type: Flow Control > Router                                          │
│                                                                      │
│ Route 1: "New Job"                                                   │
│   Condition: {{length(7.results)}} = 0                               │
│   → MODULE 9                                                         │
│                                                                      │
│ Route 2: "Duplicate — Skip"                                          │
│   Condition: {{length(7.results)}} > 0                               │
│   → (no module, execution ends for this bundle)                      │
└──────────┬──────────────────────────────────────────────────────────┘
           │ (Route 1 only)
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 9: Notion — Create Page                                       │
│ Type: Notion > Create a Database Item                                │
│ Database: Job Listings                                               │
│ Properties:                                                          │
│   Position (Title)  = {{item.title}}                                 │
│   Company           = {{item.company_name}}                          │
│   Place             = {{item.location}}                              │
│   Posting URL       = {{item.related_links[1].link}}                 │
│   Status            = "Found"                                        │
│ Output: created page ID                                              │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 10: Set Variable — Fit Score (Step 3)                         │
│ Type: Tools > Set Variable                                           │
│ Variable: fit_score                                                  │
│ Value:                                                               │
│   {{if(contains(lower(item.title); "automation"); 30; 0)             │
│   + if(contains(lower(item.title); "make"); 25; 0)                   │
│   + if(contains(lower(item.title); "consultant"); 15; 0)             │
│   + if(contains(lower(item.location); "switzerland"); 15; 0)         │
│   + 15}}                                                             │
│ Output: {{fit_score}} (number 0-100)                                 │
└──────────┬──────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│ MODULE 11: Notion — Update Page (Set Score)                          │
│ Type: Notion > Update a Database Item                                │
│ Page ID: {{9.id}}                                                    │
│ Properties:                                                          │
│   Fit Score = {{fit_score}}                                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Module Summary Table

| #  | Module                 | Type          | Purpose                        | Ops |
|----|------------------------|---------------|--------------------------------|-----|
| 1  | Scheduler              | Trigger       | Start scenario on schedule     | 0   |
| 2  | Set Variable           | Tools         | Define query list              | 1   |
| 3  | Iterator (queries)     | Flow Control  | Loop over queries              | 0   |
| 4  | HTTP Request           | HTTP          | Call SerpAPI                   | 1   |
| 5  | Iterator (jobs)        | Flow Control  | Loop over job results          | 0   |
| 6  | Filter                 | Flow Control  | Drop jobs without valid URL    | 0   |
| 7  | Notion Search          | Notion        | Dedup check by Posting URL     | 1   |
| 8  | Router                 | Flow Control  | Branch: new vs duplicate       | 0   |
| 9  | Notion Create Page     | Notion        | Write new job to database      | 1   |
| 10 | Set Variable (score)   | Tools         | Calculate Fit Score            | 1   |
| 11 | Notion Update Page     | Notion        | Write score to job record      | 1   |

**Ops per new job:** ~5 (HTTP + Search + Create + Set + Update)
**Ops per duplicate:** ~3 (HTTP + Search + Router stop)

---

## Error Handling Modules (see Failure & Retry Strategy)

Each critical module (4, 7, 9, 11) should have an **error handler** route attached. See `docs/failure-retry-strategy.md` for details.

---

## Connection Requirements

| Connection   | Service  | Used By       | Auth Type |
|-------------|----------|---------------|-----------|
| SerpAPI     | SerpAPI  | Module 4      | API Key   |
| Notion      | Notion   | Modules 7,9,11| OAuth     |

---

## How to Rebuild This Scenario

1. Create a new scenario in Make
2. Add modules in order (1 → 11)
3. Configure each module per the specs above
4. Connect SerpAPI and Notion integrations
5. Set scheduler to every 6 hours
6. Test with a single query first, then expand the query list
