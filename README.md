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

```
Scheduled Trigger (Make, daily 09:05)
  → HTTP Module (SerpAPI – Google Jobs)
    → Iterator (1 job = 1 bundle, 10 per run)
      → Notion: Create Database Item (metadata fields)
        → Notion: Append Page Content (full job description as paragraph)
```

---

## Make Scenario — 4 Modules

| # | Module | Type | Purpose |
|---|--------|------|---------|
| 4 | HTTP – Make a request | SerpAPI call | Fetches 10 job listings per query |
| 8 | Iterator | Flow Control | Splits `jobs_results[]` into individual bundles |
| 10 | Notion – Create Database Item | Notion (Legacy) | Creates a row with metadata fields |
| 17 | Notion – Append Page Content | Notion | Writes full job description into the page body |

### SerpAPI Query Parameters

| Parameter | Value |
|-----------|-------|
| engine | `google_jobs` |
| q | `Technical Product Manager` |
| location | `New York, NY, United States` |
| gl | `us` |
| hl | `en` |

Returns 10 results per call. Pagination available via `start` parameter (e.g., `start=10` for next page).

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
- [x] Make scenario created (4 modules)
- [x] HTTP ingestion via SerpAPI working
- [x] Iterator correctly splitting 10 bundles
- [x] Notion Create Database Item with all metadata fields
- [x] Notion Append Page Content with full job description
- [x] End-to-end run: 10 jobs ingested with descriptions

**Step 1 is complete and stable.**

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
