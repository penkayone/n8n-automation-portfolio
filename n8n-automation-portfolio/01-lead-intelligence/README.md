# Enterprise AI Lead Intelligence & Outreach System

I built this workflow to solve a real problem — sales teams waste hours manually qualifying leads, researching companies, and writing outreach emails. This system does all three automatically in under 10 seconds.

## What it does

A lead comes in through a webhook (from a website form, CRM, or any API). The system:

1. Validates the data and catches garbage inputs before they pollute the pipeline
2. Scores the lead 0-100 based on 12 weighted factors
3. Calls AI to analyze buying intent, confidence level, and pain points
4. Calls AI again to research the company from the email domain
5. Calls AI a third time to write a personalized outreach message based on everything it learned
6. Routes the lead to the right team with the right SLA based on tier
7. Sends a full intelligence dossier to Telegram
8. Logs everything for audit

The whole thing runs in ~7 seconds end to end.

## Why I built it this way

Most lead scoring systems are basic — they check if the email is business or personal, maybe look at the source. That's not enough. I wanted something that actually thinks about the lead the way a senior sales rep would.

The three-stage AI pipeline is the key design decision. Instead of one big prompt that tries to do everything (and does nothing well), I split it into three specialized calls:

- **Intent Analysis** — focused prompt that only figures out what the lead wants and how ready they are to buy
- **Company Research** — separate prompt that builds a company profile from the domain name
- **Outreach Generation** — takes everything from the first two calls and writes a message that actually references the lead's specific situation

Each AI call has its own error handling with retry and fallback. If one fails, the pipeline continues with defaults instead of crashing.

## Architecture

```
Webhook → Validation & Scoring → AI Intent → AI Company Research 
→ AI Outreach Generation → Priority Router (P0/P1/P2/P3) 
→ Tier-Specific Formatting → Telegram → Audit Log → API Response
```

16 nodes total. 3 AI calls. 4-way routing. Full error handling.

## Lead Scoring

The scoring engine checks 12 factors with different weights:

| Factor | Points | Logic |
|--------|--------|-------|
| Business email domain | +25 | Checks against 12 known free email providers |
| Company name provided | +15 | Direct input or extracted from email domain |
| Phone number | +10 | Validated format (min 7 digits) |
| Budget $50K+ | +20 | Parsed from string, handles various formats |
| Budget $10K+ | +15 | Mid-market tier |
| Budget $1K+ | +8 | SMB tier |
| Job title provided | +5 | Any title indicates serious inquiry |
| Detailed message (50+ chars) | +10 | Shows engagement level |
| Short message (10+ chars) | +5 | Better than nothing |
| Premium source (referral/partner) | +15 | Highest conversion sources |
| Paid source (linkedin/google_ads) | +8 | Qualified traffic |
| Has timeline | +5 | Indicates urgency |
| Large team (50+) | +5 | Enterprise signal |

Score maps to tiers:
- **70-100 → ENTERPRISE (P0)** — senior AE, 1 hour SLA
- **50-69 → QUALIFIED (P1)** — sales team, 4 hour SLA
- **25-49 → NURTURE (P2)** — SDR, 24 hour SLA
- **0-24 → COLD (P3)** — marketing automation, 48 hour SLA

## Tech stack

- **n8n** — workflow orchestration, node connections, execution engine
- **Webhook** — HTTP POST endpoint with JSON validation and response
- **JavaScript (Code node)** — data validation, scoring algorithm, JSON parsing, error handling
- **REST API (HTTP Request)** — POST to Groq/OpenAI-compatible endpoint with auth headers
- **LLM / AI** — three separate prompts with different temperatures and token limits
- **Prompt Engineering** — structured JSON output format, role-based system prompts
- **Switch node** — 4-way conditional routing based on lead tier
- **Telegram Bot API** — formatted notifications with different detail levels per tier
- **Error Handling** — retry on fail (2 attempts, 3s delay), continueOnFail for graceful degradation
- **Respond to Webhook** — structured JSON API response with lead_id, score, tier, assignment

## How to test

Import the workflow, then send a POST request to the webhook URL:

```json
{
  "name": "Maria Schmidt",
  "email": "maria@siemens.com",
  "company": "Siemens AG",
  "budget": 50000,
  "source": "referral",
  "job_title": "Head of Digital",
  "message": "Need enterprise automation for manufacturing",
  "industry": "Manufacturing",
  "team_size": "500"
}
```

Enterprise lead — score 90+, full dossier in Telegram within 8 seconds.

Try a cold lead too:

```json
{
  "name": "Test",
  "email": "test@gmail.com"
}
```

Score ~0, minimal log, marketing auto-sequence.

## What I'd add in production

- Database storage (PostgreSQL) for lead history and dedup
- Rate limiting on the webhook endpoint
- Webhook signature verification (HMAC)
- CRM push (HubSpot/Salesforce API) after scoring
- Email sequence trigger based on tier
- Dashboard metrics via Grafana
- A/B testing different outreach prompts

## Files

- `workflow.json` — ready to import into n8n
- `README.md` — this file
