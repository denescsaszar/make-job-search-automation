# Multi-Query Ingestion

## Problem

A single search query (e.g. `"automation consultant Switzerland"`) only captures a narrow slice of relevant jobs. To maximize coverage, the scenario must run **multiple queries per execution**.

---

## Strategy

Use a **Make Array → Iterator** pattern to loop over a predefined list of search queries, each hitting SerpAPI independently.

---

## Make Implementation

```
Scheduler (every 6h)
  → Set Variable: query_list (JSON array)
  → Iterator (1 query = 1 bundle)
      → HTTP Request (SerpAPI, query = {{current_query}})
      → Iterator (1 job = 1 bundle)
          → Filter (valid https:// URL)
          → Notion Search (dedup by Posting URL)
          → Router
              ├─ New → Notion: Create Page
              └─ Exists → Skip
```

---

## Query List Format

Define the query list as a JSON array in a **Set Variable** module at the start of the scenario:

```json
[
  {
    "q": "automation consultant",
    "location": "Switzerland",
    "engine": "google_jobs"
  },
  {
    "q": "make.com integrations",
    "location": "Remote",
    "engine": "google_jobs"
  },
  {
    "q": "API integration specialist",
    "location": "Zurich",
    "engine": "google_jobs"
  },
  {
    "q": "workflow automation engineer",
    "location": "Switzerland",
    "engine": "google_jobs"
  },
  {
    "q": "no-code automation",
    "location": "Europe",
    "engine": "google_jobs"
  }
]
```

---

## Field Mapping per Query

| SerpAPI Parameter | Source          | Notes                          |
|-------------------|----------------|--------------------------------|
| `q`               | `{{item.q}}`   | Search term from query list    |
| `location`        | `{{item.location}}` | Geographic filter         |
| `engine`          | `{{item.engine}}`   | Always `google_jobs` for now |
| `api_key`         | Connection      | Stored in Make connection, not in array |

---

## Why This Approach

- **Separation of concerns**: Query definitions live in one place (the Set Variable module), not scattered across duplicate HTTP modules.
- **Easy to extend**: Adding a new query = adding one JSON object to the array.
- **Dedup handles overlap**: Multiple queries may return the same job — the Posting URL dedup (Step 1) catches this automatically.
- **No extra API cost**: SerpAPI charges per search, not per result — same cost whether you use 1 or 5 queries.

---

## Scaling Considerations

| Queries | Jobs/query (avg) | Total jobs/run | Notion writes (after dedup) |
|---------|-------------------|----------------|-----------------------------|
| 1       | ~10               | ~10            | ~10                         |
| 5       | ~10               | ~50            | ~30-40 (overlap filtered)   |
| 10      | ~10               | ~100           | ~50-70                      |

Make's free tier allows 1,000 operations/month. Each query run uses ~3-5 ops per job (HTTP + Iterator + Search + Router + Create). Plan accordingly.

---

## Future: Source-Specific Queries

When additional sources are added (LinkedIn, Indeed, jobs.ch), the query list can be extended with source-specific parameters:

```json
{
  "q": "automation consultant",
  "location": "Switzerland",
  "engine": "google_jobs",
  "source": "serpapi"
}
```

The outer iterator would then route to different HTTP modules based on `{{item.source}}`.
