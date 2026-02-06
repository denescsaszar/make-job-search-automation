# 🧠 Make Job Search Automation

A production-minded job discovery and ingestion system built with **Make**, **Notion**, and **search aggregation APIs**.

This project implements the **ingestion layer** of a larger job-automation ecosystem. Its sole responsibility is to **reliably collect, validate, and persist job listings** into a shared Notion database that can be consumed by downstream agents (scoring, notifications, application tracking).

---

## 🚀 Why this project exists

Manual job searching is noisy, repetitive, and error-prone.

This automation solves that by:

- Aggregating job listings from multiple sources
- Validating data before persistence
- Normalizing inconsistent job structures
- Providing a clean, shared data layer for automation agents

The focus is **workflow design, data integrity, and scalability** — not scraping hacks.

---

## 🏗️ High-Level Architecture

Scheduled Trigger (Make)  
→ Job Aggregation (Search APIs)  
→ Iterator (1 job = 1 bundle)  
→ Validation & Filtering  
→ Notion (Shared Data Layer)

---

## ✅ STEP 1 — Job Ingestion → Notion (COMPLETED)

### Goal

Build a **stable and reliable ingestion pipeline** that writes clean job data into Notion.

No AI, no scoring, no notifications — just correct data.

---

### Implemented Make Scenario

HTTP (Search API)  
→ Iterator  
→ Filter (valid URL required)  
→ Notion (Create Page)

Each job is processed independently, ensuring partial failures never corrupt the dataset.

---

### Job Source (Current)

- SerpAPI – Google Jobs
  - Engine: google_jobs
  - Location-aware queries

Direct scraping of job boards is intentionally avoided to ensure stability and professionalism.

---

## 🧩 Shared Notion Database (Single Source of Truth)

### Fields populated in Step 1

- Position (Title)
- Company (Rich text)
- Place (Rich text)
- Posting URL (URL)
- Status (Static: Found)
- Created At (Notion auto)

---

### Data Validation Rules

- A job **must** have a valid https:// Posting URL
- Jobs without valid URLs are filtered before Notion
- No fallback URLs are written

This prevents broken records and downstream failures.

---

## 📊 Current Status

- Repository initialized ✅
- Make scenario created ✅
- HTTP ingestion working ✅
- Iterator configured correctly ✅
- URL validation filter applied ✅
- Notion database connected ✅

**Step 1 is complete and stable.**

---

## 🛠️ Tech Stack

- Make
- Notion
- SerpAPI
- GitHub

---

## 🗺️ Roadmap

- Step 2: Deduplication
- Step 3: Match Scoring
- Step 4: Notifications
- Step 5: Application Agent

---

## 💬 How to describe this project

“I designed and implemented the ingestion layer of a job discovery system using Make and Notion, focusing on data integrity, validation, and modular workflow design.”

---

## ❤️ Philosophy

This is not a toy automation.  
It’s a **production-minded system**, built with clear ownership boundaries and strict validation.
