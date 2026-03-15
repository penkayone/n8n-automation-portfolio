# n8n Automation Portfolio

Four production-grade automation systems built with n8n, each solving a different business problem with a different architectural pattern. Every workflow is tested, documented, and ready to import.

## The Workflows

### 01 — [Enterprise AI Lead Intelligence & Outreach System]

A lead comes in through a webhook — the system validates it against 20+ rules, scores it 0-100 across 12 weighted factors, then runs three separate AI calls: intent analysis, company research, and personalized outreach generation. Leads get routed by tier (Enterprise/Qualified/Nurture/Cold) with automatic team assignment and SLA. Full intelligence dossier lands in Telegram within 8 seconds.

**16 nodes · 3 AI calls · Lead scoring · Priority routing · Audit trail**

---

### 02 — [Competitive Intelligence & Market Monitor]

Runs every 12 hours. Pulls articles from 3 RSS sources, scores them through a weighted multi-group NLP keyword engine (5 keyword groups, title vs body weighting, source weighting), deduplicates across sources using fuzzy title matching, then sends the top articles to AI for strategic trend analysis. Delivers an executive briefing to Telegram with keyword heatmaps, competitor alerts, and actionable insights.

**16 nodes · 3 RSS sources · NLP scoring · Deduplication · AI analysis · Competitor alerts**

---

### 03 — [AI Customer Support & Ticket Intelligence Bot]

An interactive Telegram bot that handles customer support. Parses incoming messages, detects commands vs support requests, classifies tickets with AI (7 categories, 4 urgency levels, sentiment detection), computes SLA from a urgency-category matrix, assigns to the right team, generates an empathetic AI response, replies to the customer, and sends a full internal ticket alert to the support team. All in under 4 seconds.

**15 nodes · 2 AI calls · Interactive bot · SLA matrix · Ticket classification · Dual notifications**

---

### 04 — [Multi-API Company Research & Enrichment Pipeline]

Takes a domain name and builds a complete company profile by querying 3 DNS APIs (MX, TXT, A records), fetching the website homepage, and extracting metadata from HTML. Detects 40+ technologies from DNS records and HTML signatures — email provider, hosting, SaaS services, web frameworks, analytics tools. Computes a tech sophistication score and sends everything to AI for company profiling with sales recommendations.

**15 nodes · 3 DNS API calls · Website scraping · 40+ tech signatures · AI profiling · Tech scoring**

---

## Tech Stack

| Technology | WF#1 | WF#2 | WF#3 | WF#4 |
|-----------|:----:|:----:|:----:|:----:|
| n8n workflow design | ✓ | ✓ | ✓ | ✓ |
| Webhook (HTTP POST) | ✓ | | | ✓ |
| Schedule Trigger (Cron) | | ✓ | | |
| Telegram Bot Trigger | | | ✓ | |
| Telegram Bot API | ✓ | ✓ | ✓ | ✓ |
| REST API integration | ✓ | ✓ | ✓ | ✓ |
| JavaScript (Code node) | ✓ | ✓ | ✓ | ✓ |
| AI / LLM (Groq) | ✓ | ✓ | ✓ | ✓ |
| Prompt engineering | ✓ | ✓ | ✓ | ✓ |
| JSON parsing | ✓ | ✓ | ✓ | ✓ |
| Switch / conditional routing | ✓ | ✓ | ✓ | |
| Error handling (retry, fallback) | ✓ | ✓ | ✓ | ✓ |
| Lead scoring algorithm | ✓ | | | |
| NLP keyword engine | | ✓ | | |
| Fuzzy deduplication | | ✓ | | |
| RSS multi-source ingestion | | ✓ | | |
| SLA matrix computation | | | ✓ | |
| Ticket classification | | | ✓ | |
| Entity extraction | ✓ | | ✓ | |
| Fan-out / split pattern | | ✓ | | |
| Command routing | | | ✓ | |
| Dual notification paths | | ✓ | ✓ | |
| Audit logging | ✓ | ✓ | ✓ | ✓ |
| DNS API querying | | | | ✓ |
| Website scraping & HTML parsing | | | | ✓ |
| Tech stack detection (40+ signatures) | | | | ✓ |
| Multi-API data merging | | | | ✓ |
| Tech scoring algorithm | | | | ✓ |
| Respond to Webhook (JSON API) | ✓ | | | ✓ |

## How Each Workflow Is Different

I deliberately built each workflow around a different trigger type, data pattern, and business domain to demonstrate breadth:

- **WF#1** is request-response — a webhook receives one payload and processes it through a linear AI pipeline. The challenge is multi-stage AI orchestration where each call builds on the previous one.

- **WF#2** is scheduled batch processing — a cron job pulls data from multiple sources, aggregates it, and synthesizes insights. The challenge is multi-source data handling, deduplication, and making sense of noisy data.

- **WF#3** is interactive real-time — a live Telegram bot responding to users within seconds. The challenge is command routing, balancing AI quality with response speed, and dual-path notifications (customer + internal team).

- **WF#4** is multi-API data enrichment — multiple external API calls combined with web scraping to build a unified data profile. The challenge is orchestrating 4 different data sources (3 DNS + 1 website), parsing heterogeneous response formats, and synthesizing them into actionable intelligence.

## Quick Start

Each workflow folder has its own README with detailed setup instructions. The short version:

1. Open the `.json` file in a text editor
2. Replace `YOUR_GROQ_KEY_HERE` with your Groq API key (free at console.groq.com)
3. Replace `YOUR_CHAT_ID_HERE` with your Telegram chat ID
4. Import into n8n (Workflows → Import from File)
5. Connect your Telegram Bot credential to the Telegram nodes
6. Run it

WF#1: Send a POST request to the webhook URL with lead data.
WF#2: Just click Execute — it pulls RSS feeds automatically.
WF#3: Click Execute, then message your Telegram bot.
WF#4: Send a POST request with a company domain.

## About

Built by penkayone — automation engineer focused on n8n workflow development, API integrations, and AI-powered business process automation.

Open to freelance projects and full-time remote opportunities.

Reach out:
- Telegram: [@antongoloskokov](https://t.me/antongoloskokov)
- Email: An.goloskokov@gmail.com
