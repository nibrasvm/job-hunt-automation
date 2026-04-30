# job-hunt-automation
An n8n automation system that scrapes LinkedIn jobs, scores them with Claude AI, and generates tailored resumes and cover letters — fully integrated with Gmail and Notion.

# 🤖 Job Hunt Automation System

> Built with n8n · SerpAPI · Claude AI · Snov.io · Notion · Gmail

An end-to-end automation system that handles the full job application pipeline — from scraping Google Jobs daily to sending personalized recruiter outreach — with zero manual effort per application.

---

## 📌 Why I Built This

Job hunting is repetitive by design. As an IT professional learning automation,
I built this system to solve a real problem: spending hours every day manually
searching, evaluating, and applying. This pipeline does all of that — automatically.

---

## ⚙️ System Architecture

| # | Workflow | What It Does | Tools |
|---|---|---|---|
| W1 | **Job Scraper** | Runs 3 parallel Google Jobs searches daily, deduplicates, saves to Notion, sends email digest | SerpAPI → n8n → Notion → Gmail |
| W2 | **AI Scorer** | Scores each job 0–100 against my profile with verdict and breakdown | Notion → Claude API → Notion |
| W3 | **Resume Tailor** | Rewrites resume bullet points to match each Good Fit job | Notion → Claude API → Notion |
| W4 | **Cover Letter Gen** | Drafts personalized cover letters for tailored roles | Notion → Claude API → Notion |
| W5 | **Recruiter Finder** | Derives company domain, finds recruiter emails via Snov.io | Notion → Snov.io API → Notion |
| W6 | **Email Outreach** | Sends Claude-drafted outreach emails to recruiters daily at 10:30 AM | Notion → Claude API → Gmail → Notion |

---

## 🔁 How It Works — End to End

1. **W1 (8AM daily):** SerpAPI searches Google Jobs for 3 queries in parallel
   (NOC Engineer, Network Support Engineer, IT Support — UAE/GCC)
2. Results are merged, deduplicated, and saved to a **Notion database**
3. An email digest is sent with all new listings
4. **W2** queries Notion for unscored jobs → Claude scores each one (0–100)
   with role fit, location, experience, and skills breakdown → verdict written back to Notion
5. **W3** picks `Good Fit` jobs → Claude rewrites resume bullets tailored to each JD
6. **W4** picks `Tailored` jobs → Claude generates a full cover letter → saved to Notion
7. **W5** derives company domain from job URL or company name → Snov.io API finds
   recruiter emails → HR/talent contacts saved to Notion
8. **W6 (10:30AM daily):** Loops over `Outreach Status = Ready` rows →
   Claude drafts a personalized email → Gmail sends it → Notion updated to `Email Sent`

---

## 🛠️ Tech Stack

- **n8n** — Workflow automation engine
- **SerpAPI** — Google Jobs search API (3 parallel queries per run)
- **Anthropic Claude API** — AI scoring, resume tailoring, cover letter and email generation
- **Snov.io** — Recruiter email discovery via company domain
- **Notion API** — Central job tracking database (single source of truth)
- **Gmail** — Daily digest delivery + automated recruiter outreach

---

## 📊 Notion Database Fields

| Field | Type | Set By |
|---|---|---|
| Job Title, Company, Location, Job Link | Text/URL | W1 |
| Job Description | Text | W1 |
| AI Score (0–100), Score Breakdown, AI Verdict | Number/Text/Select | W2 |
| Tailored Resume | Text | W3 |
| Cover Letter | Text | W4 |
| Recruiter Email | Email | W5 |
| Outreach Status, Outreach Date | Select/Date | W6 |

---

## 🚀 How to Replicate This

See [`docs/setup-guide.md`](docs/setup-guide.md) for full instructions.

**Requirements:**
- n8n instance (cloud or self-hosted)
- SerpAPI key (free tier: 100 searches/month)
- Anthropic API key
- Snov.io account (free tier available)
- Notion account + API integration
- Gmail account

---

## 📎 Related

- 🔗 [LinkedIn Post — Job Scraper Workflow](https://www.linkedin.com/posts/nibrasvm_n8n-automation-jobsearch-ugcPost-7449015963473289216-T9xO?utm_source=share&utm_medium=member_desktop&rcm=ACoAAC-69AMBy61gH9o8LTtjB4wG7ojANrWNego)
- 🔗 [LinkedIn Post — AI Scoring Workflow](https://www.linkedin.com/posts/nibrasvm_ai-automation-n8n-ugcPost-7449309047155146752-DT-5?utm_source=share&utm_medium=member_desktop&rcm=ACoAAC-69AMBy61gH9o8LTtjB4wG7ojANrWNego)

---

## 👤 Author

**Nibras Hassan V M**  
CCNA Certified · IT Professional · Dubai, UAE  
[LinkedIn](https://linkedin.com/in/nibrasvm) · [GitHub](https://github.com/nibrasvm)
