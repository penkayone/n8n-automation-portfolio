# Competitive Intelligence & Market Monitor

I got tired of manually checking tech news every morning trying to figure out what competitors are doing and what's happening in the automation market. So I built a pipeline that does it for me — pulls articles from multiple sources, scores them by relevance, catches competitor mentions, runs everything through AI for strategic analysis, and drops a briefing into my Telegram. Twice a day, zero manual work.

## What it actually does

Every 12 hours the system wakes up and runs through this chain:

1. Pulls the latest articles from 3 RSS feeds — HackerNews (developer community), TechCrunch (startup news), The Verge (general tech)
2. Runs each article through a keyword scoring engine that checks 5 keyword groups with different weights
3. Throws out duplicates — the same story often appears on multiple sources, the dedup engine catches that
4. Computes analytics: keyword frequency heatmap, group breakdown, competitor mention count, source distribution
5. Sends the top articles plus analytics to AI for a structured strategic analysis
6. Formats everything into an executive briefing and pushes to Telegram
7. Checks if any direct competitor (Zapier, Make.com, etc.) was mentioned — if yes, fires a separate urgent alert

I wake up, the briefing is already in Telegram. Takes about 4-5 seconds to run the whole thing.

## How the keyword scoring works

This isn't a simple "does the article contain this word" filter. I built a weighted multi-group scoring system because not all keyword matches are equal.

There are 5 keyword groups, each with its own weight:

| Group | Keywords | Weight | Alert? |
|-------|----------|--------|--------|
| Direct competitors | zapier, make.com, n8n, workato, tray.io | 3.0x | Yes |
| Core technology | workflow automation, iPaaS, integration platform | 2.5x | No |
| AI trends | ai agent, langchain, rag, vector database, llm | 2.0x | No |
| Market signals | no-code, low-code, api integration | 1.5x | No |
| Funding signals | series a, raised, acquisition, funding | 1.8x | No |

The scoring logic stacks three multipliers:

- Keyword found in article **title** = 3 points (titles tell you what the article is really about)
- Keyword found in article **body** = 1 point (might just be a passing mention)
- Multiplied by **group weight** (competitor mention matters more than a generic "no-code" reference)
- Multiplied by **source weight** (HackerNews is 1.5x because developer sentiment is a leading indicator)

So if TechCrunch publishes "Zapier raises Series C for AI automation platform" — the scoring goes:
- "zapier" in title: 3 points x 3.0 (competitor weight) x 1.2 (TechCrunch weight) = 10.8
- "automation platform" in title: 3 x 2.5 x 1.2 = 9.0
- "ai" in body: 1 x 2.0 x 1.2 = 2.4
- Total: ~22.2 — that's a top-priority signal

Articles below 2.0 relevance score get dropped. No noise in the briefing.

## Deduplication

When the same story hits HackerNews AND TechCrunch (happens all the time with big news), I don't want to see it twice. The dedup engine compares article titles word by word, filters out short words (under 4 characters), and calculates overlap percentage. If two titles share more than 60% of their significant words, the lower-scored one gets dropped.

It's not perfect — won't catch completely rewritten headlines about the same event. But it handles the common case where three outlets publish essentially the same story with minor title variations. Good enough for a monitoring system, and it's fast.

## The AI analysis

After filtering and dedup, the top articles get packaged into a structured prompt for Groq (Llama 3.3 70B). I specifically ask for four things:

- **Market trends** — what 2-3 patterns are emerging from these articles
- **Competitive moves** — did any competitor get mentioned, and what did they do
- **Opportunities** — anything we could act on
- **Threat level** — LOW/MEDIUM/HIGH with reasoning

The prompt tells the AI to reference specific article numbers and avoid generic observations. This makes a huge difference — instead of "the market is growing" I get "article 3 suggests competitor X is moving into the Y space which could impact Z."

## Architecture

```
Schedule (12h) 
  → Config Engine (sources, keywords, weights)
    → Split Into Feeds (fan-out to 3 sources)
      → Fetch RSS (20 articles each)
        → NLP Keyword Scoring (weighted multi-group)
          → Aggregate & Deduplicate (merge + fuzzy dedup)
            → Has Data? 
              ├── [no matches] → "Nothing found" notification
              └── [has data] → Build AI Prompt 
                  → AI Trend Analysis (Groq LLM)
                    → Build Executive Brief
                      → Send to Telegram
                      → Competitor Alert Check
                        → Has Competitor?
                          ├── [yes] → Send Urgent Alert
                          └── [no] → Execution Log
```

16 nodes. 3 RSS sources. 5 keyword groups. AI analysis. Dual notification path (briefing + competitor alert). Full error handling on RSS fetches.

## Tech stack

- **n8n** — workflow orchestration
- **Schedule Trigger** — cron-based, fires every 12 hours automatically
- **RSS Feed Read** — multi-source news ingestion from 3 different feeds
- **JavaScript (Code node)** — NLP keyword engine with weighted scoring, fuzzy deduplication algorithm, analytics computation, prompt construction, data normalization
- **Fan-out pattern** — splits config into individual feed tasks for parallel-style processing
- **Data aggregation** — merges results from multiple sources, deduplicates, computes cross-source analytics
- **REST API (HTTP Request POST)** — call to Groq API with structured JSON body and auth headers
- **AI / LLM** — Llama 3.3 70B via Groq for strategic market analysis
- **Prompt engineering** — role-based analyst prompt with specific output format (trends, competitive moves, opportunities, threat level)
- **IF node** — conditional branching for empty results and competitor alert routing
- **Telegram Bot API** — two separate notification paths: main briefing and urgent competitor alert
- **Error handling** — continueOnFail on RSS (feeds timeout sometimes), retry on AI calls

## How to set up

**Step 1:** Open the JSON file in a text editor (TextEdit on Mac, Notepad on Windows)

**Step 2:** Find and replace (Cmd+H or Ctrl+H):
- Find `YOUR_GROQ_KEY_HERE` → replace with your Groq API key (starts with `gsk_`)
- Find `YOUR_CHAT_ID_HERE` → replace with your Telegram chat ID (a number). This appears twice — once for the main briefing and once for the competitor alert.

**Step 3:** Import the JSON file into n8n (Workflows → + → Import from File)

**Step 4:** Click on both Telegram nodes (12. Send to Telegram and 15. Send Competitor Alert) and select your Telegram credential from the dropdown

**Step 5:** Click "Execute Workflow" — no webhook needed, it runs immediately. Wait 5-10 seconds. Check Telegram for the intelligence briefing.

## What I'd add next

- More data sources — Reddit API for r/nocode and r/automation, Twitter/X API for real-time signals
- PostgreSQL storage for historical tracking — plot keyword frequency over weeks and months
- Weekly comparison report — "this week vs last week" trend delta
- Slack integration alongside Telegram for team distribution
- Custom keyword groups per client or project
- Sentiment analysis per article instead of just keyword presence
- Grafana dashboard with charts showing market trends over time

## Design decisions worth explaining

**Why one AI call here but three in the Lead Intelligence workflow?**

Different problem shape. Lead intelligence needs multiple perspectives on a single entity — intent analysis is a different skill than company research, which is different from writing outreach copy. Three specialized prompts beat one big prompt there. But market monitoring is the opposite — I need one big-picture view across many articles. A single well-structured prompt with all the data works better for synthesis and pattern recognition.

**Why RSS instead of web scraping?**

Three reasons: RSS is reliable (no HTML structure changes breaking your selectors), legal (it's a public feed meant for consumption), and lightweight (just XML, no browser rendering). For a system that runs automatically twice a day, I need something that won't require maintenance every time a website redesigns. Scraping is great for one-off data collection, but RSS is built for monitoring.

**Why weighted scoring instead of simple keyword counting?**

Because "Zapier" in a headline is a completely different signal than "zapier" mentioned once in a paragraph listing 20 automation tools. And a funding article about a direct competitor is way more important than a generic "top 10 no-code tools" listicle. The weight system captures these nuances. It took some tuning to get the weights right, but now the top-ranked articles are consistently the ones I'd actually want to read.

## Files

- `workflow.json` — ready to import into n8n
- `README.md` — this file
