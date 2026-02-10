# Roadmap

## Step 1 — Job Ingestion (COMPLETED)

- [x] Make scenario with SerpAPI integration
- [x] Iterator pattern (1 job = 1 bundle)
- [x] URL validation filter
- [x] Notion database write (Create + Append Page Content for full descriptions)
- [x] Field mappings: Position, Place, Company, Posting URL, Status, Job Description (page body)
- [x] Documentation: [step-1-ingestion.md](step-1-ingestion.md)

## Step 1.5 — Pagination (IN PROGRESS)

- [ ] Rebuild scenario with Repeater → HTTP → Iterator → Notion → Set Variable loop
- [ ] Use SerpAPI `next_page_token` for token-based pagination (10 jobs/page)
- [ ] Add stop filter to prevent wasted API credits on last page
- [ ] Documentation: [pagination.md](pagination.md)

## Step 2 — Deduplication & Multi-Query

- [ ] Check `share_link` in Notion before creating (prevent daily duplicates)
- [ ] Add multi-query ingestion (array of search terms + locations)
- [ ] Add error handlers to all critical modules
- [ ] Add rate limiting (Sleep modules)
- [ ] Add email alerting on scenario failure
- [ ] Documentation: [multi-query-ingestion.md](multi-query-ingestion.md), [failure-retry-strategy.md](failure-retry-strategy.md)

## Step 3 — Fit Score

- [ ] Add rule-based scoring (weighted keyword match)
- [ ] Add Fit Score + Score Tier fields to Notion
- [ ] Configure keyword lists in Set Variable module
- [ ] Documentation: [fit-score-logic.md](fit-score-logic.md)

## Step 4 — Notifications

- [ ] Trigger email/Slack for High-tier jobs (score >= 70)
- [ ] Weekly digest for Medium-tier jobs
- [ ] Configure notification channels

## Step 5 — Application Agent

- [ ] Track application status in Notion
- [ ] Auto-apply or generate cover letters (optional)
- [ ] Follow-up reminders
