# 🏗️ Architecture Overview

This document describes the high-level architecture of the **Make Job Search Automation**.

The system is designed as a **reliable ingestion layer** that collects job data,
normalizes it, and stores it in a shared Notion database for downstream agents
(scoring, application, notifications).

---

## 🔄 High-Level Flow

```
┌──────────────┐
│   Scheduler  │
│   (Make)     │
└──────┬───────┘
       │
       ▼
┌────────────────────┐
│ Job Source Ingest  │
│ (HTTP / Search API)│
│  - Google Jobs     │
│  - LinkedIn        │
│  - Indeed          │
│  - jobs.ch         │
└──────┬─────────────┘
       │
       ▼
┌────────────────────┐
│ Normalization Layer│
│  - title           │
│  - company         │
│  - location        │
│  - job URL         │
│  - description     │
└──────┬─────────────┘
       │
       ▼
┌────────────────────┐
│ Filtering & Guards │
│  - required fields │
│  - valid URLs      │
│  - location rules  │
│  - keyword filters │
└──────┬─────────────┘
       │
       ▼
┌────────────────────┐
│ Iterator (Make)    │
│  - one job / bundle│
└──────┬─────────────┘
       │
       ▼
┌────────────────────┐
│ Deduplication      │
│  - job URL hash    │
│  - title + company │
│    heuristic       │
└──────┬─────────────┘
       │
       ▼
┌────────────────────┐
│ Notion Database    │
│ (Shared Data Layer)│
│  - ingestion-owned │
│  - agent-owned     │
└──────┬─────────────┘
       │
       ▼
┌────────────────────┐
│ Downstream Agents  │
│ (out of scope)     │
│  - scoring         │
│  - notifications   │
│  - auto-apply      │
└────────────────────┘
```

---

## 🧩 Ownership Model

```
Ingestion Automation (this repo)
├─ Job discovery
├─ Data normalization
├─ Validation & safety
└─ Notion upsert

Decision / Action Agents (future)
├─ Match scoring
├─ Status changes
├─ Applications
└─ Follow-ups
```

---

## 🛡️ Design Principles

- No brittle scraping
- Schema-safe Notion writes
- Fail fast on invalid data
- One job = one bundle
- Production-minded, interview-ready

---

## 📚 Detailed Documentation

| Document | Purpose |
|----------|---------|
| [Step 1: Ingestion](step-1-ingestion.md) | Ingestion pipeline + deduplication by Posting URL |
| [Data Model](data-model.md) | Field-level data flow diagram, ownership rules |
| [Multi-Query Ingestion](multi-query-ingestion.md) | Running multiple search queries per execution |
| [Fit Score Logic](fit-score-logic.md) | Rule-based 0–100 scoring against target profile |
| [Make Scenario Mapping](make-scenario-mapping.md) | Module-by-module Make scenario reference |
| [Failure & Retry Strategy](failure-retry-strategy.md) | Error handling, retries, rate limits, alerting |
| [Roadmap](roadmap.md) | Step-by-step implementation plan |
