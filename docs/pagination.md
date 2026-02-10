# Pagination — Token-Based (SerpAPI Google Jobs)

## Problem
SerpAPI Google Jobs returns 10 results per page. The `start` offset parameter is **deprecated** for this engine. Pagination now uses `next_page_token` from the `serpapi_pagination` object in each response.

## Solution: Repeater + Set Variable Loop

### Module Chain
```
Repeater → HTTP → Iterator → Notion Create → Notion Append → Set Variable
```

### Module Details

| # | Module | Config |
|---|--------|--------|
| 1 | **Repeater** (Flow Control) | Initial value: 1, Repeats: 10 (max 100 jobs) |
| 2 | **HTTP – Make a request** | SerpAPI URL + all params + `next_page_token` from Set Variable |
| 3 | **Iterator** (Flow Control) | Array: `{{2.data.jobs_results}}` |
| 4 | **Notion – Create Database Item** | Position, Place, Company, Posting URL, Status |
| 5 | **Notion – Append Page Content** | Page ID from [4], Paragraph text: `{{3.description}}` |
| 6 | **Tools – Set Variable** | name: `next_page_token`, value: `{{2.data.serpapi_pagination.next_page_token}}` |

### How It Works
- **Iteration 1**: Set Variable is empty → SerpAPI ignores the empty `next_page_token` param → returns page 1
- **Iteration 2+**: Set Variable has the token from previous response → SerpAPI returns the next page
- **Last page**: No `next_page_token` in response → variable becomes empty → subsequent iterations return the last page again

### Stop Filter (Recommended)
Between Repeater and HTTP, add a filter to prevent wasting API credits after the last page:
- **Condition**: Repeater `{{1.i}}` = 1 **OR** `{{6.next_page_token}}` is not empty

### API Credits
- Each page = 1 SerpAPI credit (10 jobs)
- 10 pages max = 10 credits per run
- Daily schedule = ~300 credits/month

## SerpAPI Response Structure
```json
{
  "jobs_results": [ ... 10 jobs ... ],
  "serpapi_pagination": {
    "next_page_token": "eyJzdGFydCI6MTB9...",
    "next": "https://serpapi.com/search.json?engine=google_jobs&next_page_token=..."
  }
}
```

## Rebuild Steps
1. Delete all existing modules in the scenario
2. Add Repeater as the first module (it inherits the schedule trigger)
3. Add HTTP after Repeater — same URL/params + `next_page_token` query param
4. Add Iterator after HTTP — array: `jobs_results`
5. Add Notion Create after Iterator — same field mappings
6. Add Notion Append after Create — Page ID + description paragraph
7. Add Set Variable (Tools) after Append — store `next_page_token`
8. Run once to test
9. Re-enable daily schedule
