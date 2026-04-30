# System Architecture — Job Hunt Automation

This document explains the design decisions behind the system,
how the workflows connect, and why each tool was chosen.

---

## Design Philosophy

The system is built around a single principle: **every manual step
in a job hunt is a repeatable process that can be automated.**

Rather than building one monolithic workflow, the system is split into
7 independent workflows chained through a central Notion database.
Each workflow does one thing well, writes its output back to Notion,
and triggers the next stage by updating a status field.

This design means:
- Any workflow can be debugged or rerun independently
- If one stage fails, the others are unaffected
- New workflows can be added at any stage without restructuring

---

## The Central Hub — Notion Database

Notion acts as the single source of truth for the entire pipeline.
Every workflow reads from it and writes back to it.

The database uses status fields as pipeline gates:
Scoring Status:      Pending → Scored
Resume Status:       Pending → Tailored
Cover Letter Status: Pending → Cover Letter Ready
Outreach Status:     Ready → Email Sent → Followed Up
Each workflow filters by the status it expects upstream, processes
the matching rows, and updates the status to unlock the next stage.
This means the pipeline is self-regulating — no manual handoff needed.

---

## Workflow Chain

W1 — Job Scraper (8:00 AM)
↓ writes: Job Title, Company, Location, Description, Job Link
↓ sets:   Scoring Status = Pending

W2 — AI Scorer (8:30 AM)
↓ reads:  Scoring Status = Pending
↓ writes: AI Score, AI Verdict, Score Breakdown
↓ sets:   Scoring Status = Scored

W3 — Resume Tailor (9:00 AM)
↓ reads:  AI Verdict = Good Fit, Resume Status = Pending
↓ writes: Tailored Resume
↓ sets:   Resume Status = Tailored

W4 — Cover Letter Generator (9:30 AM)
↓ reads:  Resume Status = Tailored, Cover Letter Status ≠ Ready
↓ writes: Cover Letter
↓ sets:   Cover Letter Status = Cover Letter Ready

W5 — Recruiter Finder (10:00 AM)
↓ reads:  Cover Letter Status = Cover Letter Ready, Recruiter Email = empty
↓ writes: Recruiter Name, Recruiter Email, Recruiter Position
↓ sets:   Outreach Status = Ready

W6 — Email Outreach (10:30 AM)
↓ reads:  Outreach Status = Ready
↓ writes: sends Gmail outreach
↓ sets:   Outreach Status = Email Sent, Outreach Date

W7 — Follow-up Tracker (11:00 AM)
↓ reads:  Outreach Status = Email Sent, Outreach Date > 5 days ago
↓ writes: sends Gmail follow-up
↓ sets:   Outreach Status = Followed Up

---

## Tool Selection Rationale

**SerpAPI — Job Discovery**
LinkedIn's Jobs API is not publicly available. SerpAPI provides
structured access to Google Jobs results, which aggregates listings
from multiple job boards. Three parallel searches run simultaneously
(NOC Engineer, Network Support Engineer, IT Support Engineer) to
maximize coverage without multiplying runtime.

**Notion — Pipeline Database**
Chosen over a spreadsheet or SQL database for two reasons: the visual
Kanban/table view makes it easy to manually review and intervene at
any stage, and the Notion API is straightforward to integrate with n8n.

**Claude API (Haiku model) — AI Processing**
Used across W2, W3, W4, W6, and W7 for different tasks: structured
JSON scoring, resume rewriting, cover letter generation, and email
drafting. The Haiku model was chosen specifically for speed and
cost-efficiency — it handles high-volume repetitive tasks across
multiple workflows daily without significant API cost.

**Snov.io — Recruiter Discovery**
Given a company domain, Snov.io returns verified professional email
addresses with position titles and confidence scores. W5 derives the
domain from the job URL or company name, then filters results by HR
and recruiting keywords to surface the most relevant contact.

**Gmail — Outreach Delivery**
Used for both the daily job digest (W1) and recruiter outreach (W6, W7).
All outbound emails are BCC'd to a personal address for personal tracking.
W6 and W7 include a 30-second wait between sends to avoid triggering
Gmail's rate limits.

---

## Deduplication Strategy

W1 pulls the full existing Notion database before writing new jobs,
extracts all existing Job Link URLs into a Set, then filters incoming
SerpAPI results against that Set. Only jobs with URLs not already in
Notion are written. This prevents duplicate entries across daily runs
without needing a separate deduplication workflow.

---

## Error Handling Approach

- W2 and W3 initialize a result object before the API call so a
  fallback value is always returned even if Claude errors
- W4 checks that the cover letter exceeds 50 characters before writing
  to Notion, discarding empty or failed responses
- W5 falls through to a No Operation node if Snov.io returns no emails,
  leaving the row untouched for manual review
- W6 checks for a non-empty Recruiter Email before calling Claude —
  rows without an email are marked Not Ready and skipped

---

## Schedule Design

Workflows are staggered at 30-minute intervals starting at 8:00 AM
Dubai time. This ensures each workflow's upstream data is fully written
to Notion before the next one reads it, without requiring explicit
workflow-to-workflow triggers.

| Time | Workflow |
|---|---|
| 8:00 AM | W1 — Job Scraper |
| 8:30 AM | W2 — AI Scorer |
| 9:00 AM | W3 — Resume Tailor |
| 9:30 AM | W4 — Cover Letter Generator |
| 10:00 AM | W5 — Recruiter Finder |
| 10:30 AM | W6 — Email Outreach |
| 11:00 AM | W7 — Follow-up Tracker |
