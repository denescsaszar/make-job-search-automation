# Step 1 — Job Ingestion (Completed)

## Scope

Reliably collect job listings from SerpAPI Google Jobs and persist full records to Notion, including complete job descriptions in the page body.

---

## Make Scenario Flow (4 Modules)

```
HTTP [4] (Trigger, scheduled daily 09:05 Europe/Berlin)
  → Iterator [8] (1 job = 1 bundle, 10 per page)
  → Notion [10]: Create Database Item (metadata in properties)
  → Notion [17]: Append Page Content (full description in page body)
```

Scenario ID: **4388094** on eu1.make.com (org 988735)

---

## Fields Written

| Notion Property | Source              | Type      | Notes |
|----------------|---------------------|-----------|-------|
| Position       | `8.title`           | Title     | Job title from SerpAPI |
| Company        | `8.company_name`    | Rich text | Employer name |
| Place          | `8.location`        | Rich text | City/state |
| Posting URL    | `8.share_link`      | URL       | Google Jobs share link |
| Status         | `"To-do: Found"`    | Select    | Static value for all new jobs |
| Contact        | (empty)             | Rich text | Reserved for manual entry |

### Page Body (via Append Page Content)

| Content        | Source              | Type      | Notes |
|---------------|---------------------|-----------|-------|
| Job Description | `8.description`   | Paragraph | Full text, no 2000-char limit |

The two-step write pattern (Create + Append) is used because Notion rich text properties are limited to 2000 characters. Job descriptions from SerpAPI are often 3000-5000+ characters. Writing the description as page body content bypasses this limit entirely.

---

## SerpAPI Configuration

| Parameter  | Value |
|-----------|-------|
| URL       | `https://serpapi.com/search.json` |
| engine    | `google_jobs` |
| q         | `Technical Product Manager` |
| location  | `New York, NY, United States` |
| gl        | `us` |
| hl        | `en` |
| api_key   | (stored in HTTP module) |

Returns 10 results per page. See [pagination.md](pagination.md) for multi-page fetching.

---

## Key Design Decisions

- **share_link** used as Posting URL (not `related_links`) — more reliable, always present
- **Append Page Content** for descriptions — bypasses 2000-char rich text limit
- **Iterator Array** must use a data-picker pill (`4.data.jobs_results[]`), not literal text
- **No URL filter currently active** — share_link is always a valid Google URL
- **No deduplication yet** — planned for Step 2, critical before enabling daily schedule

---

## Status

- Ingestion pipeline: **working** (10 jobs per run)
- Job descriptions: **full text in page body**
- Pagination: **not yet implemented** (Step 1.5)
- Deduplication: **not yet implemented** (Step 2)
