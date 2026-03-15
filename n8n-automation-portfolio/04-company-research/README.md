# Multi-API Company Research & Enrichment Pipeline

Every time a new lead comes in, someone on the sales team has to Google the company, check what tools they use, figure out how big they are, and decide how to approach them. That's 15-20 minutes per lead, and most of it can be automated. So I built a pipeline that takes a domain name and builds a full company profile in 8 seconds.

## What it does

You give it a domain — `hubspot.com`, `stripe.com`, whatever. The system:

1. Queries Google DNS API for MX records — figures out what email provider the company uses (Google Workspace, Microsoft 365, Zoho, etc.)
2. Queries Google DNS API for TXT records — discovers which SaaS services the company has verified (HubSpot, Salesforce, Stripe, DocuSign, Atlassian, Shopify, and 15+ others leave traces in DNS)
3. Queries Google DNS API for A records — identifies the hosting provider from IP ranges (AWS, Google Cloud, Cloudflare, Vercel, Azure)
4. Fetches the actual website homepage and parses the HTML
5. Extracts metadata (title, description, Open Graph data)
6. Detects 20+ web technologies from HTML signatures (React, WordPress, Next.js, Google Analytics, Intercom, Tailwind, etc.)
7. Merges all data sources and computes a tech sophistication score (0-100)
8. Sends everything to AI for company profiling — size estimate, industry, budget indicator, sales approach recommendation
9. Builds a formatted research report and sends to Telegram + returns as JSON API response

One domain in, full company dossier out. No manual research needed.

## Why DNS is surprisingly useful for company research

Most people think of DNS as "the thing that turns domain names into IP addresses." But DNS records are actually a goldmine of business intelligence that nobody thinks to check.

**MX records** tell you what email system a company uses. A company on Google Workspace is different from one on Microsoft 365 — it hints at their tech culture, budget, and decision-making patterns. Enterprise email security (Mimecast, Barracuda) signals a larger organization with compliance requirements.

**TXT records** are the real treasure. Every time a company sets up Salesforce, verifies their domain with Facebook Business, connects Stripe, or configures DocuSign — they add a TXT record. It's like reading a company's tech stack from their DNS. I check for 17 different service signatures.

**A records** (IP addresses) reveal hosting infrastructure. Specific IP ranges map to AWS, Google Cloud, Azure, Cloudflare, Vercel, GitHub Pages, and Fastly. A company on AWS with Cloudflare CDN has a very different infrastructure profile than one on shared hosting.

## Tech scoring algorithm

Not all tech stacks are equal. I built a scoring system that estimates how technically sophisticated a company is:

| Factor | Points | Logic |
|--------|--------|-------|
| Has email provider detected | +15 | Any known provider |
| Enterprise email security | +10 | Mimecast, Barracuda |
| 5+ SaaS services in DNS | +25 | Heavy SaaS adoption |
| 3-4 SaaS services | +15 | Moderate adoption |
| 1-2 SaaS services | +8 | Basic setup |
| AWS or Google Cloud hosting | +15 | Enterprise infrastructure |
| Cloudflare | +10 | Modern CDN |
| HubSpot or Salesforce | +10 | CRM = serious about sales |
| Stripe or Shopify | +10 | Payment/commerce signals |
| 3+ web technologies detected | +15 | Modern web stack |
| 1-2 web technologies | +8 | Basic web presence |
| Website loads successfully | +5 | Active domain |

Score maps to levels:
- **60-100 → Enterprise** — large org, multiple SaaS tools, modern infrastructure
- **35-59 → Advanced** — solid tech adoption, likely mid-market
- **18-34 → Intermediate** — some tools, growing
- **0-17 → Basic** — minimal tech footprint

HubSpot scored 90/100 in testing — Google Workspace, Cloudflare, 14 technologies detected including Stripe, Atlassian, Google Analytics, Segment, and more. That's an accurate signal for what they actually are.

## Website technology detection

After DNS, the pipeline fetches the actual homepage and scans the HTML for technology signatures. It detects 22 different technologies:

**CMS/Frameworks:** WordPress, Shopify, Wix, Squarespace, React, Next.js, Vue.js, Angular
**CSS:** Bootstrap, Tailwind CSS
**Analytics:** Google Tag Manager, Google Analytics, Hotjar, Segment
**Chat/Support:** Intercom, Zendesk, Drift, Crisp
**Business:** HubSpot, Stripe, reCAPTCHA, Cloudflare

Each detection is based on specific HTML patterns — `wp-content` for WordPress, `_next` for Next.js, `googletagmanager` for GTM. Simple regex checks but they're reliable because these tools inject predictable markup.

## Architecture

```
Webhook (POST with domain)
  → Parse Input & Extract Domain
    → DNS MX Lookup (Google DNS API)
      → DNS TXT Lookup (Google DNS API)
        → DNS A Record Lookup (Google DNS API)
          → Analyze Tech Stack (merge 3 DNS results, detect 40+ signals)
            → Fetch Website Homepage (HTTP GET)
              → Extract Website Metadata (HTML parsing, 22 tech signatures)
                → Merge All Data & Build AI Prompt
                  → AI Company Profiling (Groq LLM)
                    → Parse AI Profile
                      → Build Research Report
                        → Send to Telegram + Research Log
                          → API JSON Response
```

15 nodes. 3 DNS API calls. 1 website fetch. 1 AI call. 40+ technology signatures. Sequential pipeline with full error handling.

## Tech stack

- **n8n** — workflow orchestration
- **Webhook** — POST endpoint accepting domain, email, or company name
- **HTTP Request (GET)** — 3 calls to Google DNS API (dns.google/resolve) for MX, TXT, and A record types
- **HTTP Request (GET)** — website homepage fetch with redirect following
- **JavaScript (Code node)** — domain extraction and normalization, DNS response parsing, email provider detection, hosting provider detection from IP ranges, SaaS service detection from TXT records (17 services), HTML metadata extraction with regex, web technology detection (22 technologies), tech scoring algorithm, data merging across 4 sources, Telegram text sanitization
- **REST API (HTTP Request POST)** — Groq LLM API call for AI company profiling
- **AI / LLM** — structured JSON company profile generation (size, industry, budget, sales approach)
- **Prompt engineering** — analyst role prompt with strict JSON output schema
- **Multi-source data merging** — combining DNS intelligence + website intelligence + AI synthesis into unified profile
- **Error handling** — continueOnFail on all external API calls (DNS can timeout, websites can block), retry on AI
- **Telegram Bot API** — formatted research report delivery
- **Respond to Webhook** — full JSON API response with profile, tech stack, and website data

## How to set up

**Step 1:** Open the JSON file in a text editor. Find and replace:
- `YOUR_GROQ_KEY_HERE` → your Groq API key (appears 1 time)
- `YOUR_CHAT_ID_HERE` → your Telegram chat ID (appears 1 time)

**Step 2:** Import into n8n

**Step 3:** Click on the Telegram node (13. Send to Telegram) and select your Telegram credential

**Step 4:** Click "Execute workflow" → open https://reqbin.com in another tab

**Step 5:** In reqbin:
- Method: POST
- URL: your test webhook URL (shown under node 1)
- Content: JSON
- Body:
```json
{"domain": "hubspot.com", "context": "Potential client for automation services"}
```

**Step 6:** Click Send in reqbin → check n8n for green nodes → check Telegram for the report

**Other test inputs:**
```json
{"domain": "stripe.com"}
```
```json
{"email": "john@shopify.com"}
```
```json
{"domain": "n8n.io", "company": "n8n GmbH"}
```

Each domain gives completely different results — different tech stacks, different scores, different AI profiles.

## Design decisions

**Why DNS instead of paid enrichment APIs?**

Two reasons. First, DNS data is free, public, and unlimited — no API keys, no rate limits, no monthly fees. Second, it provides ground-truth data. When I see a Salesforce TXT verification record, that company is definitely using Salesforce — it's not a guess from some third-party database that might be six months out of date. In production, I'd combine this with Clearbit or similar for employee count and revenue data, but DNS alone gives you a surprisingly complete picture.

**Why sequential DNS calls instead of parallel?**

I originally built this with parallel DNS calls (fan-out from node 2 to nodes 3, 4, 5 simultaneously). It didn't work reliably on n8n Cloud — the merge node couldn't always access results from all three branches. Sequential calls add ~200ms total overhead but work 100% reliably. For an 8-second pipeline, that's an acceptable trade-off. On self-hosted n8n with the Merge node, I'd go back to parallel.

**Why detect tech from HTML instead of using a tool like BuiltWith?**

Same reason as DNS — it's free and instant. BuiltWith and Wappalyzer APIs cost money and add latency. My 22 regex checks run in under 1ms and catch the major technologies. Not as comprehensive as BuiltWith's database of 80,000 technologies, but more than enough for a sales qualification workflow. I'm detecting the technologies that actually matter for selling automation services — CMS, frameworks, analytics, CRM, chat tools.

**Why combine DNS + website + AI instead of just asking AI about the company?**

Because AI hallucinates company data. If I ask "tell me about hubspot.com" the AI gives a decent answer, but it might be wrong about specific tech details. My approach gives AI real, verified data to work with — it's synthesizing facts, not guessing. The AI adds value by interpreting what the tech stack means for sales strategy, but the underlying data is verified from primary sources.

## What I'd add next

- Clearbit or Apollo API integration for employee count, revenue, and social profiles
- LinkedIn company page scraping for recent posts and hiring signals
- SSL certificate analysis (organization name, certificate authority)
- WHOIS data for domain age and registrant info
- Historical comparison — track tech stack changes over time in PostgreSQL
- Batch processing mode — upload a CSV of 100 domains, get 100 reports
- Chrome extension — right-click any website, get instant company profile
- Webhook-based integration with CRM — auto-enrich new contacts in HubSpot/Salesforce

## Files

- `workflow.json` — ready to import into n8n
- `README.md` — this file
