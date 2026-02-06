# Failure & Retry Strategy

## Design Principle

**One job = one bundle.** A failure on job #5 must never prevent job #6 from being processed. The iterator pattern in Make isolates each job into its own execution bundle, so errors are contained.

---

## Failure Modes & Handling

### 1. SerpAPI Failures (Module 4: HTTP Request)

| Failure           | Cause                          | Handling                              |
|-------------------|--------------------------------|---------------------------------------|
| 401 Unauthorized  | Invalid or expired API key     | Stop scenario, alert via email        |
| 429 Rate Limited  | Too many requests              | Retry after 60s (Make auto-retry)     |
| 500 Server Error  | SerpAPI outage                 | Retry 3x with exponential backoff     |
| Timeout           | Slow response                  | Retry 2x, then skip query             |
| Empty results     | No jobs found for query        | Continue to next query (not an error) |

**Make config:**
- Enable "Auto retry" on the HTTP module
- Max retries: 3
- Initial interval: 60 seconds

---

### 2. Notion Search Failures (Module 7: Dedup Check)

| Failure            | Cause                         | Handling                              |
|--------------------|-------------------------------|---------------------------------------|
| 401 Unauthorized   | Notion token expired          | Stop scenario, alert via email        |
| 429 Rate Limited   | Notion API rate limit (3 req/s)| Retry with backoff                   |
| 502/503 Server     | Notion outage                 | Retry 3x, then skip job              |
| Timeout            | Slow query                    | Retry 2x, then skip job              |

**Important:** If the dedup search fails, **do not proceed to Create Page**. A failed search could lead to duplicate writes. The error handler should route to "skip" rather than "create".

---

### 3. Notion Create Page Failures (Module 9)

| Failure              | Cause                       | Handling                             |
|----------------------|-----------------------------|--------------------------------------|
| 400 Validation Error | Bad field value / type       | Log error, skip job, continue        |
| 401 Unauthorized     | Token expired                | Stop scenario, alert                 |
| 429 Rate Limited     | Notion rate limit            | Retry with backoff                   |
| 409 Conflict         | Duplicate (race condition)   | Ignore — dedup will catch on next run|
| 502/503 Server       | Notion outage                | Retry 3x, then skip job             |

---

### 4. Notion Update Page Failures (Module 11: Fit Score)

| Failure              | Cause                       | Handling                             |
|----------------------|-----------------------------|--------------------------------------|
| 400 Validation Error | Bad score value              | Log, skip — job exists without score |
| 404 Not Found        | Page deleted between ops     | Log, skip — race condition           |
| 429 Rate Limited     | Notion rate limit            | Retry with backoff                   |

**Non-critical:** If the score update fails, the job record still exists in Notion. Score can be recalculated on the next run or manually.

---

## Make Error Handler Configuration

Attach error handlers to modules 4, 7, 9, and 11:

```
Module (e.g., HTTP Request)
  ├─ Success → continue normal flow
  └─ Error Handler →
       ├─ Retry (for 429, 5xx): Resume with backoff
       ├─ Ignore (for empty results): Continue to next bundle
       └─ Break (for 401, persistent failures): Stop scenario
```

### Error Handler Types in Make

| Handler  | Behavior                                         | Use When                  |
|----------|--------------------------------------------------|---------------------------|
| Resume   | Retry the failed module                          | Transient errors (429, 5xx)|
| Ignore   | Skip this bundle, continue with next             | Non-critical failures      |
| Break    | Stop the entire scenario execution               | Auth failures, config bugs |
| Commit   | Save partial results, then stop                  | Data integrity required    |
| Rollback | Discard partial results, then stop               | Atomic operations needed   |

### Recommended Setup

| Module              | Handler | Retry Count | Backoff     | Fallback    |
|---------------------|---------|-------------|-------------|-------------|
| HTTP Request (4)    | Resume  | 3           | 60s / 120s / 240s | Ignore (skip query) |
| Notion Search (7)   | Resume  | 2           | 30s / 60s   | Ignore (skip job)*  |
| Notion Create (9)   | Resume  | 3           | 30s / 60s / 120s | Ignore (skip job) |
| Notion Update (11)  | Resume  | 2           | 30s / 60s   | Ignore (skip score) |

*When dedup search fails, **skip the job entirely** — do not fall through to Create Page.

---

## Rate Limiting Strategy

### Notion API Limits

- **3 requests per second** per integration
- With multi-query ingestion (5 queries x 10 jobs), worst case = 50 Notion Search + 50 Create = 100 API calls
- At 3 req/s, that's ~33 seconds minimum execution time

**Mitigation:** Add a **Sleep** module (1 second) between the Notion Search and Notion Create modules to stay safely under the rate limit.

```
Notion Search → Sleep (1s) → Router → Notion Create → Sleep (1s) → Notion Update
```

### SerpAPI Limits

- Depends on plan (100 searches/month on free tier)
- 5 queries x 4 runs/day = 20 searches/day = 600/month — **needs paid plan**
- Consider reducing to 2 runs/day (10 searches/day = 300/month)

---

## Alerting

### When to Alert

| Event                     | Alert Method      | Urgency |
|---------------------------|-------------------|---------|
| Auth failure (401)        | Email (Make)      | High    |
| Scenario stopped by Break | Email (Make)      | High    |
| 10+ consecutive skips     | Email (Make)      | Medium  |
| SerpAPI quota exhausted   | Email (Make)      | Medium  |
| Notion outage (3+ retries)| Email (Make)      | Low     |

### Make Email Alert Setup

Add a **Mailgun / Gmail** module on the Break error handler path:

```
Error Handler (Break)
  → Send Email
      To: your@email.com
      Subject: "[Job Automation] Scenario Failed — {{error.message}}"
      Body: "Module: {{error.moduleName}}, Error: {{error.message}}, Time: {{now}}"
```

---

## Recovery Procedures

### After Auth Failure (401)

1. Reconnect the affected integration in Make (SerpAPI or Notion)
2. Test the connection
3. Re-enable the scenario
4. Run manually once to verify

### After Prolonged Outage

1. Check Make execution history for failed runs
2. Note the last successful run timestamp
3. Run the scenario manually to catch up
4. Dedup will prevent duplicates from the catch-up run

### After Schema Change (Notion)

1. If a Notion field is renamed/deleted, the Create/Update modules will fail with 400
2. Update the module mappings in Make to match the new schema
3. Test with a single job before re-enabling the schedule

---

## Monitoring Checklist

- [ ] Check Make execution history weekly
- [ ] Verify Notion database row count is growing
- [ ] Check SerpAPI usage dashboard monthly
- [ ] Review error handler logs for patterns
- [ ] Test scenario manually after any config change
