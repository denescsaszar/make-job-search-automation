# 🧠 Make Job Search Automation

An automated job discovery and matching system built with **Make**, **Notion**, and **search aggregation APIs**.

This project continuously collects job listings from multiple platforms, normalizes and scores them, stores them in a shared Notion database, and notifies when high-fit roles appear — forming the ingestion layer of a larger job-application agent system.

---

## 🚀 Why this project exists

Searching for jobs manually is noisy, repetitive, and inefficient.

This automation solves that by:

- Aggregating job listings from **multiple sources**
- Normalizing inconsistent job data
- Scoring roles based on profile fit
- Acting as a **reliable, reusable data pipeline** for downstream agents

The focus is **workflow design, data quality, and decision automation** — not scraping hacks.

---

## 🏗️ High-Level Architecture

**Schedule Trigger (Make)**  
→ Job Aggregation (Search APIs / RSS)  
→ Data Normalization  
→ Filtering (Location, Keywords)  
→ Optional AI Match Scoring  
→ Deduplication  
→ Notion (Shared Data Layer)  
→ Notifications (Email / Slack)

This automation is designed to run **in parallel** with a separate job-application agent that consumes the same Notion database.

---

## 🧩 Data Model (Shared Notion Database)

The Notion database acts as the **single source of truth**.

### Ingestion-owned fields (this automation)

- Job Title
- Company
- Location
- Source
- Job URL
- Description
- Match Score
- Date Added
- Ingested By

### Agent-owned fields (handled elsewhere)

- Status (New / Shortlisted / Applied / Rejected)
- Applied At
- CV Version
- Notes
- Agent Decision

Clear ownership prevents conflicts and allows multiple systems to evolve independently.

---

## 🔍 Job Sources

Jobs are collected via **aggregation and search APIs**, including listings originating from:

- LinkedIn
- Indeed
- jobs.ch
- Company career pages

Direct scraping of job boards is intentionally avoided to ensure:

- Stability
- Legal safety
- Interview-ready professionalism

---

## 🛠️ Tech Stack

- **Make** – workflow orchestration
- **Notion** – shared data layer
- **Search / SERP APIs** – job aggregation (e.g. Google Jobs)
- **OpenAI (optional)** – match scoring
- **Email / Slack** – notifications

---

## 🧠 Match Scoring (Optional)

Job descriptions can be scored (0–100) against a predefined profile using AI.

Example criteria:

- API & integration experience
- Automation / workflow tools
- Consulting or technical account roles

Only high-match roles trigger notifications.

---

## 🧪 Current Status

- Repository initialized ✅
- Architecture defined ✅
- Notion database connected ✅
- Automation build: **in progress**

---

## 🗺️ Roadmap

- [ ] Job ingestion via search APIs
- [ ] Data normalization & filtering
- [ ] Deduplication logic
- [ ] Notion upsert
- [ ] Match scoring
- [ ] Notifications
- [ ] Error handling & retries
- [ ] Architecture diagram

---

## 💬 How to talk about this project

> “I designed a modular job discovery system using Make, with a shared Notion data layer and clear ownership boundaries between ingestion and decision-making agents. The focus was reliability, clean data flow, and automation at scale.”

---

## 📌 Next steps

1. Create the Make scenario skeleton
2. Connect the Notion database
3. Add the first job source
4. Implement deduplication early
5. Add AI scoring and notifications

---

## ❤️ Philosophy

This is not a toy project.  
It’s a **production-minded automation**, designed like a consulting deliverable and built to scale.
