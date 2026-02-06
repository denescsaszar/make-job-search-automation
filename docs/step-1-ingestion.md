# Step 1 — Job Ingestion (Completed)

## Scope

Reliably collect job listings from search APIs and persist validated records to Notion.

---

## Make Scenario Flow

```
Scheduler (every 6h)
  → HTTP Request (SerpAPI – Google Jobs)
  → Iterator (1 job = 1 bundle)
  → Filter (Posting URL starts with https://)
  → Notion: Create Page
```

---

## Fields Written

| Notion Field | Source            | Type      | Required |
|-------------|-------------------|-----------|----------|
| Position    | `title`           | Title     | Yes      |
| Company     | `company_name`    | Rich text | Yes      |
| Place       | `location`        | Rich text | No       |
| Posting URL | `related_links[0].link` | URL | Yes (filter gate) |
| Status      | static `"Found"`  | Select    | Yes      |
| Created At  | Notion auto       | Date      | Auto     |

---

## Validation Rules

1. **Posting URL must start with `https://`** — enforced by a Make filter before the Notion module.
2. Jobs that fail the filter are silently dropped (no error, no partial write).
3. No fallback URLs are written — a missing URL means the job is not persisted.

---

## Deduplication by Posting URL

### Problem

The same job listing can appear in multiple SerpAPI responses across scheduled runs. Without deduplication, the Notion database accumulates duplicate rows.

### Strategy

Use **Posting URL** as the unique key. Before creating a Notion page, check if a page with the same Posting URL already exists.

### Make Implementation

```
Iterator
  → Filter (valid https:// URL)
  → Notion: Search (filter: Posting URL = current URL)
  → Router
      ├─ Route 1: Result count = 0 → Notion: Create Page (new job)
      └─ Route 2: Result count > 0 → No action (skip duplicate)
```

### Module Config

**Notion Search module:**
- Database: Job Listings
- Filter property: `Posting URL`
- Filter condition: `equals`
- Filter value: `{{current_job.posting_url}}`

**Router conditions:**
- Route 1 (Create): `{{length(search_results)}} = 0`
- Route 2 (Skip): `{{length(search_results)}} > 0`

### Why URL-only (not title + company)?

- URLs are globally unique identifiers for a posting.
- Title + company matching is fuzzy and risks false positives (same company, similar roles).
- URL dedup is deterministic, zero false positives.
- If needed later, a secondary title+company heuristic can be layered on top.

### Edge Cases

| Scenario | Handling |
|----------|----------|
| Same job, different URL (aggregator redirect) | Treated as separate — acceptable for now |
| URL changes after repost | Treated as new job — correct behavior |
| Notion Search API latency | Make retries automatically; see Failure & Retry Strategy |

---

## Status After Step 1

- Ingestion pipeline: **stable**
- Deduplication: **active (URL-based)**
- Data quality: **validated before write**
