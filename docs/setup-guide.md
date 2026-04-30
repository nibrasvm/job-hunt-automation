# Setup Guide — Job Hunt Automation System

This guide walks you through replicating the full 6-workflow automation system
from scratch. Follow the sections in order — each workflow depends on the
infrastructure set up before it.

---

## Prerequisites

Before touching n8n, create accounts on all these platforms:

| Service | Purpose | Free Tier |
|---|---|---|
| [n8n](https://n8n.io) | Workflow automation engine | 14-day trial (or self-host free) |
| [SerpAPI](https://serpapi.com) | Google Jobs search | 100 searches/month |
| [Anthropic](https://console.anthropic.com) | Claude API (AI reasoning) | Pay-as-you-go |
| [Snov.io](https://snov.io) | Recruiter email discovery | 50 credits/month |
| [Notion](https://notion.so) | Job tracking database | Free |
| Gmail | Email digest + outreach | Free |

---

## Step 1 — Build the Notion Database

This is the central hub. Every workflow reads from and writes to this one database.

1. Create a new Notion page → click **+ New database → Table**
2. Name it: `Job Hunt Pipeline`
3. Add the following fields exactly as listed:

| Field Name | Type | Notes |
|---|---|---|
| Job Title | Title | Default field — rename it |
| Company | Text | |
| Location | Text | |
| Job Link | URL | |
| Job Description | Text | |
| Source | Text | e.g. "SerpAPI – NOC Engineer Dubai" |
| Date Added | Date | |
| Scoring Status | Select | Options: `Pending`, `Scored`, `Error` |
| AI Score | Number | 0–100 |
| Score Breakdown | Text | JSON string from Claude |
| AI Verdict | Select | Options: `🔥 Apply Now`, `✅ Good Fit`, `🟡 Maybe`, `❌ Skip` |
| Scored At | Date | |
| Resume Status | Select | Options: `Pending`, `Tailored`, `Error` |
| Tailored Resume | Text | |
| Cover Letter Status | Select | Options: `Pending`, `Generated`, `Error` |
| Cover Letter | Text | |
| Recruiter Email | Email | |
| Outreach Status | Select | Options: `Ready`, `Not Ready`, `Email Sent`, `Followed Up` |
| Outreach Date | Date | |

4. Note your **Database ID** — it appears in the URL when you open the database:
   `notion.so/YOUR-WORKSPACE/`**`3399bdd8248c80fbbe10f701e8c3d55f`**`?v=...`

---

## Step 2 — Connect Services to n8n

### Notion
1. Go to [notion.so/my-integrations](https://notion.so/my-integrations) → **New integration**
2. Name it `n8n` → copy the **Internal Integration Token**
3. Open your Notion database → **⋯ menu → Add connections → select your integration**
4. In n8n: **Credentials → New → Notion API** → paste the token

### Gmail
1. In n8n: **Credentials → New → Gmail OAuth2**
2. Follow the OAuth flow — sign in with your Google account
3. Grant permission to send and read mail

### SerpAPI
1. Sign up at [serpapi.com](https://serpapi.com) → copy your **API Key** from the dashboard
2. Used directly as an HTTP header in n8n — no dedicated credential type needed

### Anthropic Claude API
1. Go to [console.anthropic.com](https://console.anthropic.com) → **API Keys → Create Key**
2. In n8n: **Credentials → New → Header Auth**
   - Name: `Anthropic API`
   - Header Name: `x-api-key`
   - Header Value: your API key
3. Model used across all workflows: `claude-haiku-4-5-20251001`

### Snov.io
1. Go to [app.snov.io](https://app.snov.io) → **API** section → copy **Client ID** and **Client Secret**
2. Used via HTTP Request nodes in n8n (OAuth2 token flow)

---

## Step 3 — Import the Workflows

1. In n8n, click **+ New Workflow**
2. Open the **⋯ menu (top right) → Import from file**
3. Select the `.json` file from the `/workflows` folder of this repo
4. Repeat for each workflow

After importing each workflow, update the **Notion Database ID** in every
Notion node — replace the placeholder with your own database ID from Step 1.

---

## Workflow Overview & Trigger Schedule

| File | Workflow | Trigger | Runs At |
|---|---|---|---|
| `01-job-scraper.json` | Google Jobs scraper + email digest | Schedule | Daily, 8:00 AM |
| `02-ai-scoring.json` | Claude AI job scoring | Schedule | Daily, 8:30 AM |
| `03-resume-tailoring.json` | Resume bullet tailoring | Schedule | Daily, 9:00 AM |
| `04-cover-letter-gen.json` | Cover letter generation | Schedule | Daily, 9:30 AM |
| `05-recruiter-finder.json` | Snov.io recruiter email lookup | Schedule | Daily, 10:00 AM |
| `06-email-outreach.json` | Gmail recruiter outreach | Schedule | Daily, 10:30 AM |

The 30-minute stagger is intentional — each workflow depends on the
previous one having completed and written its results back to Notion.

---

## Step 4 — Configure W1 (Job Scraper)

W1 runs 3 SerpAPI searches in parallel. After importing, update the 3 HTTP
Request nodes with your own search queries:
https://serpapi.com/search.json?engine=google_jobs&q=NOC+Engineer+Dubai&location=Dubai,UAE&api_key=YOUR_KEY
https://serpapi.com/search.json?engine=google_jobs&q=Network+Support+Engineer+UAE&location=UAE&api_key=YOUR_KEY
https://serpapi.com/search.json?engine=google_jobs&q=IT+Support+Engineer+GCC&location=GCC&api_key=YOUR_KEY
Customize the `q=` parameter to match your target roles and the
`location=` parameter for your geography.

---

## Step 5 — Activate All Workflows

Once imported and configured:

1. Open each workflow
2. Toggle the **Inactive → Active** switch at the top
3. Activate in order: W1 → W2 → W3 → W4 → W5 → W6

> ⚠️ Do not activate W6 until you have verified that W5 is correctly
> populating the `Recruiter Email` field in Notion. Sending outreach to
> wrong or empty email addresses will cause errors.

---

## Troubleshooting

**Notion node returns "Could not find database"**
→ Check that your integration is connected to the database (Step 2, Notion section)
→ Verify the Database ID has no dashes — n8n uses the raw 32-character string

**SerpAPI returns 0 results**
→ Test your search URL directly in the browser first
→ Check your monthly quota at serpapi.com/dashboard

**Claude API returns markdown code fences in JSON fields**
→ Add `.replace(/```json|```/g, '').trim()` to your Code node output

**Snov.io returns no emails**
→ The company domain derived from the job URL may be incorrect
→ Check the domain field in your Notion row and correct it manually for that entry

**W6 skips rows with `Outreach Status = Not Ready`**
→ This is expected — it means Snov.io found no recruiter email for that company
→ You can manually add an email to the `Recruiter Email` field and
   change `Outreach Status` to `Ready` to include it in the next run

---

## Notes

- All workflows use `claude-haiku-4-5-20251001` — fast and cost-efficient for high-volume runs
- Snov.io free tier provides 50 credits/month; each domain lookup costs 1 credit
- SerpAPI free tier allows 100 searches/month; 3 searches/day = ~90/month (within limit)
- This system was built and tested on n8n cloud; it is fully compatible with self-hosted n8n
