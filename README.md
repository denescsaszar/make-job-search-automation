# Make Job Search Automation

A production-minded job discovery and ingestion system built with **Make**, **Notion**, and **SerpAPI**.

This project implements the **ingestion layer** of a larger job-automation ecosystem. Its sole responsibility is to **reliably collect, validate, and persist job listings** into a shared Notion database that can be consumed by downstream agents (scoring, notifications, application tracking).

---

## Why this project exists

Manual job searching is noisy, repetitive, and error-prone.

This automation solves that by:

- Aggregating job listings from multiple sources via search APIs
- Iterating each result as an independent bundle for fault isolation
- Persisting structured job data into a shared Notion database
- Appending full job descriptions as page content (bypassing Notion's 2000-char property limit)

The focus is **workflow design, data integrity, and scalability** — not scraping hacks.

---

## High-Level Architecture

### v1 — Single-page ingestion (Scenario 4388094, stable)
```
Scheduled Trigger (Make, daily 09:05)
  → HTTP Module (SerpAPI – Google Jobs)
    → Iterator (1 job = 1 bundle, 10 per run)
      → Notion: Create Database Item (metadata fields)
        → Notion: Append Page Content (full job description as paragraph)
```

### v2 — Paginated ingestion (Scenario 4426581, in progress)
```
Scheduled Trigger
  → Repeater (N pages)
    → HTTP Module (SerpAPI + next_page_token via get())
      → Iterator (10 jobs per page)
        → Notion: Search Objects (dedup check)
          → Notion: Create Database Item
            → Notion: Append Page Content
    → Set Variable (store next_page_token for next iteration)
```

---

## Make Scenarios

### v1 — Single Page (Scenario 4388094, stable)

| # | Module | Type | Purpose |
|---|--------|------|---------|
| 4 | HTTP – Make a request | SerpAPI call | Fetches 10 job listings per query |
| 8 | Iterator | Flow Control | Splits `jobs_results[]` into individual bundles |
| 10 | Notion – Create Database Item | Notion (Legacy) | Creates a row with metadata fields |
| 17 | Notion – Append Page Content | Notion | Writes full job description into the page body |

### v2 — Paginated (Scenario 4426581, in progress)

| # | Module | Type | Purpose |
|---|--------|------|---------|
| 1 | Repeater | Flow Control | Loop N times (one per page) |
| 2 | HTTP – Make a request | SerpAPI call | Fetches 10 jobs + next_page_token |
| 3 | Iterator | Flow Control | Splits `jobs_results[]` into bundles |
| 4 | Notion – Search Objects | Notion | Dedup check by share_link |
| 5 | Notion – Create Database Item | Notion (Legacy) | Creates row with metadata |
| 6 | Notion – Append Page Content | Notion | Full description in page body |
| 7 | Tools – Set Variable | Tools | Stores `next_page_token` for next iteration |

### SerpAPI Query Parameters

| Parameter | Value |
|-----------|-------|
| engine | `google_jobs` |
| q | `Technical Project Manager` |
| location | `Berlin, Germany` |
| gl | `de` |
| hl | `en` |
| api_key | (stored in module) |

Returns 10 results per call. Pagination uses `next_page_token` from `serpapi_pagination` object (the `start` offset param is deprecated for Google Jobs).

---

## Notion Database Schema

### Fields populated by Module 10 (Create Database Item)

| Notion Field | Maps to | Type |
|-------------|---------|------|
| Position | `8.title` | Title |
| Place | `8.location` | Rich text |
| Company | `8.company_name` | Rich text |
| Posting URL | `8.share_link` | URL |
| Status | `To-do: Found` (static) | Select |
| Contact | (empty, reserved) | Rich text |

### Page body populated by Module 17 (Append Page Content)

| Content | Maps to | Type |
|---------|---------|------|
| Paragraph block | `8.description` | Rich text (full length) |

The Append module receives the `Database Item ID` from Module 10 and writes the complete job description into the page body as a paragraph block. This avoids the 2000-character limit on Notion rich text properties.

---

## Key Design Decisions

**Iterator with proper pill references** — The Iterator Array field uses a data-picker pill (`4.data.jobs_results[]`), not literal text. This ensures Make correctly splits the API response into individual bundles.

**Two-step Notion write** — Metadata goes into database properties (Create), while the full description goes into page content (Append). This separation keeps the database view clean while preserving complete job details.

**No description in properties** — Job descriptions from SerpAPI average 3000-5000 characters, exceeding Notion's 2000-char rich text property limit. The Append Page Content module has no such limit.

**Fault isolation** — Each job is processed as an independent bundle. If one Notion write fails, the remaining jobs still succeed.

---

## Current Status

- [x] Repository initialized
- [x] v1 scenario created (4 modules, single page)
- [x] HTTP ingestion via SerpAPI working
- [x] Iterator correctly splitting 10 bundles
- [x] Notion Create + Append Page Content with full descriptions
- [x] End-to-end run: 10 jobs ingested with descriptions
- [x] v2 scenario created with Repeater + Set Variable pagination loop
- [ ] Fix `next_page_token` conditional omission on first iteration
- [ ] End-to-end paginated run (30+ jobs)

**Step 1 complete. Step 1.5 (pagination) in progress.**

---

## Tech Stack

- **Make** — Workflow automation (scenario, iterator, HTTP)
- **Notion** — Shared database and page content storage
- **SerpAPI** — Google Jobs search aggregation
- **GitHub** — Version control and documentation

---

## Roadmap

- Step 2: Deduplication (prevent duplicate job entries)
- Step 3: Multi-query ingestion (multiple roles/locations)
- Step 4: Fit scoring (match jobs against profile)
- Step 5: Notifications
- Step 6: Application agent

---

## Documentation

Detailed design docs are in the [`docs/`](docs/) directory:

- [`architecture.md`](docs/architecture.md) — System architecture overview
- [`data-model.md`](docs/data-model.md) — Notion database schema and data flow
- [`step-1-ingestion.md`](docs/step-1-ingestion.md) — Step 1 implementation details
- [`multi-query-ingestion.md`](docs/multi-query-ingestion.md) — Multi-query strategy
- [`fit-score-logic.md`](docs/fit-score-logic.md) — Fit score design
- [`make-scenario-mapping.md`](docs/make-scenario-mapping.md) — Make scenario module mapping
- [`failure-retry-strategy.md`](docs/failure-retry-strategy.md) — Error handling and retries
- [`roadmap.md`](docs/roadmap.md) — Project roadmap

---

## Philosophy

This is not a toy automation.
It's a **production-minded system**, built with clear ownership boundaries and strict validation.
