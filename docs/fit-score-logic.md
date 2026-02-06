# Fit Score Logic (Step 3)

## Purpose

Assign a **0–100 Fit Score** to each ingested job, measuring how well the listing matches the user's target profile. High-scoring jobs get prioritized for review and notifications.

---

## Scoring Model

The score is a **weighted keyword match** against the job title, company, and (when available) description. No AI/LLM required for v1 — pure rule-based logic inside Make.

---

## Profile Criteria (Configurable)

Define scoring criteria as a weighted list:

| Criterion           | Weight | Match Target                        |
|---------------------|--------|-------------------------------------|
| Role keyword        | 30     | Title contains: `automation`, `integration`, `API`, `workflow` |
| Tool keyword        | 25     | Title/description contains: `Make`, `Zapier`, `n8n`, `Notion` |
| Seniority match     | 15     | Title contains: `consultant`, `specialist`, `engineer`, `lead` |
| Location preference | 15     | Place matches: `Switzerland`, `Remote`, `Zurich`              |
| Company signal      | 15     | Company in whitelist OR not in blacklist                       |

**Total possible: 100**

---

## Make Implementation

### Option A: Rule-Based (Recommended for v1)

```
Notion: Create Page (after dedup)
  → Set Variable: score = 0
  → Set Variable: score += 30 if title contains role keywords
  → Set Variable: score += 25 if title contains tool keywords
  → Set Variable: score += 15 if title contains seniority keywords
  → Set Variable: score += 15 if location matches
  → Set Variable: score += 15 if company passes check
  → Notion: Update Page (set Fit Score = {{score}})
```

In Make, this translates to a chain of **Set Variable** modules using `if()` / `contains()` functions:

```
{{if(contains(lower(title); "automation"); 30; 0)
+ if(contains(lower(title); "make"); 25; 0)
+ if(contains(lower(title); "consultant"); 15; 0)
+ if(contains(lower(place); "switzerland"); 15; 0)
+ if(contains(lower(company); "deloitte"); 0; 15)}}
```

### Option B: AI-Scored (Future, Step 3b)

```
Notion: Create Page
  → HTTP: OpenAI API (send title + description + profile)
  → Parse JSON (extract score)
  → Notion: Update Page (set Fit Score)
```

Prompt template:
```
Rate this job 0-100 for fit with this profile:
- Target: automation/integration consulting roles
- Skills: Make, APIs, Notion, workflow design
- Location: Switzerland or remote

Job: {{title}} at {{company}} in {{place}}

Return only a number 0-100.
```

---

## Notion Fields (Added by Step 3)

| Field       | Type   | Owner   | Notes                          |
|-------------|--------|---------|--------------------------------|
| Fit Score   | Number | Step 3  | 0-100, calculated on ingestion |
| Score Tier  | Formula| Notion  | `High` (70+), `Medium` (40-69), `Low` (0-39) |

**Notion formula for Score Tier:**
```
if(prop("Fit Score") >= 70, "High", if(prop("Fit Score") >= 40, "Medium", "Low"))
```

---

## Score Tiers & Actions

| Tier   | Score Range | Downstream Action               |
|--------|-------------|----------------------------------|
| High   | 70–100      | Notify immediately (Step 4)      |
| Medium | 40–69       | Add to weekly review list        |
| Low    | 0–39        | No action, stored for reference  |

---

## Keyword Lists (Editable)

Store these in a Make **Data Store** or as a **Set Variable** at scenario start for easy updates:

**Role keywords:** `automation`, `integration`, `API`, `workflow`, `process`, `digital transformation`

**Tool keywords:** `Make`, `Zapier`, `n8n`, `Notion`, `Airtable`, `Integromat`, `no-code`, `low-code`

**Seniority keywords:** `consultant`, `specialist`, `engineer`, `lead`, `senior`, `manager`, `architect`

**Location whitelist:** `Switzerland`, `Remote`, `Zurich`, `Bern`, `Basel`, `Europe`

---

## Why Rule-Based First

- Zero API cost (no OpenAI calls).
- Deterministic — same input always produces same score.
- Transparent — easy to explain and debug.
- Fast — no external HTTP call latency.
- AI scoring can be layered on later for jobs that need deeper analysis (e.g., parsing descriptions).
