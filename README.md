# 🧠 Make Job Search Automation

A production-minded job discovery and ingestion system built with **Make**, **Notion**, and **search aggregation APIs**.

This project implements the **ingestion layer** of a larger job-automation ecosystem. Its sole responsibility is to **reliably collect, validate, and persist job listings** into a shared Notion database that can be consumed by downstream agents (scoring, notifications, application tracking).

---

## 🚀 Why this project exists

Manual job searching is:

- Noisy
- Repetitive
- Error-prone

This automation solves that by:

- Aggregating job listings from multiple sources
- Validating data before persistence
- Normalizing inconsistent job structures
- Providing a clean, shared data layer for automation agents

The focus is **workflow design, data integrity, and scalability** — not scraping hacks.

---

## 🏗️ High-Level Architecture
