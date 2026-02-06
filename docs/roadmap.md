# Roadmap

## Step 1 — Job Ingestion (COMPLETED)

- [x] Make scenario with SerpAPI integration
- [x] Iterator pattern (1 job = 1 bundle)
- [x] URL validation filter
- [x] Notion database write
- [x] Deduplication by Posting URL
- [x] Documentation: [step-1-ingestion.md](step-1-ingestion.md)

## Step 2 — Multi-Query & Hardening

- [ ] Add multi-query ingestion (array of search terms)
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
