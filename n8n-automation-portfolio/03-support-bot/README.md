# AI Customer Support & Ticket Intelligence Bot

Most support bots are glorified FAQ menus — press 1 for billing, press 2 for technical. I wanted something that actually understands what the customer is saying, figures out how urgent it is, writes a real response, and routes the ticket to the right team with proper SLA. So I built one.

## What it does

Someone writes to the Telegram bot — anything from "my payment was charged twice" to "how do I reset my password" to just "/help". The system figures out what to do:

If it's a command (/start, /help, /status, /feedback) — instant system response, no AI needed.

If it's a support request — the real pipeline kicks in:

1. Parses the message, extracts metadata (length, language, whether it contains URLs, emails, phone numbers)
2. Sends to AI for classification — determines category (billing, technical, bug, complaint, feature request, account, general), urgency level, customer sentiment, and whether a human needs to get involved
3. Computes SLA from a matrix — billing + critical = 30 minutes, feature request + low = 24 hours. Every combination has a specific deadline.
4. Assigns to the right team — billing goes to finance, bugs go to engineering, complaints go to customer success
5. Sends to AI again to generate an empathetic, context-aware response for the customer
6. Replies to the customer in Telegram with the auto-generated response plus ticket info
7. Simultaneously sends a full internal alert to the support team with classification, SLA, the original message, and the AI response that was sent

If it's feedback — logs it and thanks the user.

The whole thing takes about 3-4 seconds. Customer gets a real answer, not "your ticket has been received."

## The classification engine

I use AI for classification instead of keyword matching because customer messages are messy. Someone writing "this is ridiculous, I've been waiting 3 weeks for my invoice to be corrected" isn't going to type the word "billing" — but AI understands that's a billing complaint with high urgency and angry sentiment.

The AI returns structured JSON with 7 fields:

```
category: billing | technical | account | feature_request | bug_report | general_inquiry | complaint
urgency: critical | high | medium | low
sentiment: angry | frustrated | neutral | positive
requires_human: true/false
summary: one sentence
extracted_product: product name if mentioned
extracted_error: error description if mentioned
language: detected language code
```

Temperature is set to 0.1 — I need consistent, predictable classification, not creative writing. That comes later in the response generation step.

## SLA matrix

This is one of the things I'm most proud of in this workflow. Instead of one-size-fits-all "we'll respond within 24 hours," every ticket gets a specific SLA based on both urgency AND category:

| | Critical | High | Medium | Low |
|---|---------|------|--------|-----|
| Billing | 30 min | 1 hr | 4 hr | 8 hr |
| Technical | 15 min | 30 min | 2 hr | 8 hr |
| Bug report | 15 min | 30 min | 2 hr | 8 hr |
| Complaint | 30 min | 1 hr | 4 hr | 8 hr |
| Account | 30 min | 1 hr | 4 hr | 8 hr |
| Feature request | 2 hr | 4 hr | 8 hr | 24 hr |
| General inquiry | 1 hr | 2 hr | 8 hr | 24 hr |

A critical billing issue gets 30 minutes. A low-priority feature request gets 24 hours. The SLA deadline timestamp is calculated and included in both the customer reply and the internal alert.

## Priority scoring

On top of the tier-based routing, each ticket gets a numeric priority score (0-100) that factors in urgency AND sentiment:

- Critical urgency: 100 base
- High: 75
- Medium: 50
- Low: 25
- Angry customer: +20
- Frustrated customer: +10

So a high-urgency ticket from an angry customer scores 95 — that goes to the top of the queue. A low-priority neutral inquiry scores 25 — it'll get handled, but not before the fires are out.

## Team assignment

Each category routes to a specific team:

| Category | Team |
|----------|------|
| billing | finance_team |
| technical | tech_support_l2 |
| bug_report | engineering |
| complaint | customer_success |
| account | account_management |
| feature_request | product_team |
| general_inquiry | support_l1 |

## Architecture

```
Telegram Bot Trigger 
  → Message Parser (extract metadata, detect commands)
    → Route Message Type (Switch)
      ├── [support request] → AI Classify → Enrich & SLA 
      │     → AI Generate Response → Parse → Build Reply
      │       → Reply to Customer (Telegram)
      │       → Build Internal Alert → Notify Team (Telegram)
      │       → Ticket Log
      ├── [feedback] → Handle Feedback → Send Reply
      └── [command] → System Response → Send Reply
```

15 nodes. 2 AI calls. 3-way command routing. SLA matrix computation. Dual Telegram notifications (customer + internal team). Entity extraction.

## Tech stack

- **n8n** — workflow orchestration
- **Telegram Bot Trigger** — interactive bot that receives messages in real-time (not webhook, not schedule — live trigger)
- **JavaScript (Code node)** — message parsing, command detection, metadata extraction (URL/email/phone regex), SLA matrix computation, priority scoring, team assignment, Telegram text cleanup
- **Switch node** — 3-way routing: support requests → AI pipeline, feedback → handler, commands → system responses
- **REST API (HTTP Request POST)** — two calls to Groq API with different prompts and temperatures
- **AI ticket classification** — structured JSON output with category, urgency, sentiment, entity extraction (temperature 0.1 for consistency)
- **AI response generation** — empathetic customer reply matched to detected language and context (temperature 0.6 for natural tone)
- **Prompt engineering** — two specialized prompts: classifier (strict JSON, low temp) and responder (natural language, higher temp)
- **Telegram Bot API** — three separate Telegram nodes: customer reply (dynamic chat_id), internal team alert (fixed chat_id), system/feedback replies
- **Error handling** — retry on AI calls, continueOnFail, fallback responses if AI fails

## How to set up

**Step 1:** Open the JSON file in a text editor. Find and replace:
- `YOUR_GROQ_KEY_HERE` → your Groq API key (appears 2 times)
- `YOUR_CHAT_ID_HERE` → your Telegram chat ID (appears 1 time — for internal team alerts)

**Step 2:** Import into n8n

**Step 3:** Click on ALL THREE Telegram nodes (10, 12, 14) and select your Telegram credential

**Step 4:** Click "Execute workflow" → go to Telegram → write to your bot:
```
My payment was charged twice and I need a refund immediately. Order #12345
```

You'll receive two messages: AI-generated customer response + full internal support ticket.

**Also test:**
- `/help` — shows command list
- `/start` — welcome message  
- `/feedback Great bot!` — logs feedback
- Any text message — gets classified and answered

Note: in test mode, the bot processes one message per execution. Click Execute again for each test message.

## What I'd build next

- Conversation memory — right now each message is independent, I'd add context from previous messages in the same chat
- Database (PostgreSQL) to store all tickets with full history
- Dashboard showing ticket volume, avg response time, sentiment trends, SLA compliance rate
- Escalation workflow — if SLA deadline passes without human action, auto-escalate
- Multi-language response templates for common issues
- Integration with actual helpdesk (Zendesk, Freshdesk) via API
- Voice message support via Whisper transcription

## Design decisions

**Why two AI calls instead of one?**

Classification and response generation are fundamentally different tasks. Classification needs to be precise and consistent — I use temperature 0.1 and strict JSON format. Response generation needs to be natural and empathetic — I use temperature 0.6 and free-form text. One prompt trying to do both would compromise on both.

**Why Telegram and not a web widget?**

For the portfolio this demonstrates the same architecture that would work with any channel. Telegram is just the interface — swap it for a web widget webhook, email parser, or Slack bot and the core pipeline (classify → enrich → respond → route → notify) stays identical. I chose Telegram because it's instant to test and visually demonstrates the bot working in real-time.

**Why regex for entity detection in the parser instead of AI?**

Speed. The parser runs in <1ms. URLs, emails, and phone numbers have predictable patterns that regex handles perfectly. No reason to burn an AI call for something a regular expression does better and faster. AI is reserved for the tasks that actually need reasoning — classification and response generation.

## Files

- `workflow.json` — ready to import into n8n
- `README.md` — this file
