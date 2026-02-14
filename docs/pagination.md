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
- **Iteration 1**: `get("next_page_token")` returns empty → `next_page_token` param must be **omitted entirely** → SerpAPI returns page 1
- **Iteration 2+**: Set Variable has the token from previous response → SerpAPI returns the next page
- **Last page**: No `next_page_token` in response → variable becomes empty → subsequent iterations return the last page again

### Critical: Omitting the Token on First Request
SerpAPI returns `"Invalid next_page_token"` if the parameter is sent with an empty or invalid value. The token must be **completely absent** from the first request, not just empty.

**Approaches tried:**
1. ~~`ifempty(get("next_page_token"); ignore)`~~ — `ignore` keyword doesn't work inside `ifempty()` for HTTP query params; Make sends the literal string or empty value
2. ~~Built-in HTTP v4 pagination (Token or cursor-based)~~ — Known bug: `items path did not point to an array` even when the response structure is correct ([Make community thread](https://community.make.com/t/http-v4-pagination-items-path-incorrectly-treated-as-object-should-be-array/98601))
3. **Recommended: Build token into URL conditionally** — Use `if()` to construct the full URL with or without `&next_page_token=...`
4. **Alternative: Remove query param, use Router** — Branch on iteration number: first iteration goes to HTTP without token param, subsequent iterations go to a second HTTP module with the token param

### Stop Filter (Recommended)
Between Repeater and HTTP, add a filter to prevent wasting API credits after the last page:
- **Condition**: Repeater `{{1.i}}` = 1 **OR** `{{get("next_page_token")}}` is not empty

### API Credits
- Each page = 1 SerpAPI credit (10 jobs)
- 3 pages default = 3 credits per run (30 jobs)
- Daily schedule = ~90 credits/month

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

## Current State (v2, Scenario 4426581)

The v2 scenario has the Repeater + Set Variable loop wired up:
```
Repeater [1] → HTTP [2] → Iterator [3] → Notion Search [4] → Notion Create [5] → Notion Append [6] → Set Variable [7]
```

**Remaining issue:** The `next_page_token` query parameter must be conditionally omitted on the first Repeater iteration. SerpAPI rejects empty/invalid tokens with `"Invalid next_page_token"`.

### Next Fix
Replace the `next_page_token` query parameter approach with a dynamic URL:
- Remove `next_page_token` from the HTTP module's query parameters
- Change the URL field to: `{{if(get("next_page_token"); "https://serpapi.com/search.json?next_page_token=" + get("next_page_token"); "https://serpapi.com/search.json")}}`
- Keep all other parameters (engine, q, location, gl, hl, api_key) as separate query params — they work on both first and subsequent requests
