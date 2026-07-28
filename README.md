# Refund Request Automation (n8n + AI)

An n8n workflow that automates the triage of refund requests, classifying each customer into one of three profiles and taking a differentiated action for each — instead of treating every request the same way.

## Problem

Support teams often spend hours manually reviewing refund requests, with the added risk of treating a high-value (VIP) customer exactly the same as any other — losing revenue and goodwill in the process.

## Solution

This workflow uses an AI Agent (Google Gemini 2.5 Flash) to read each incoming refund form submission, cross-reference the customer against a Google Sheets database, and route the request through one of three paths based on purchase value and sentiment analysis of the customer's comment.

## How it works

```
Webhook (form submission)
   → AI Agent (Gemini 2.5 Flash)
       - looks up the customer in Google Sheets (as a tool)
       - calculates days since purchase
       - runs sentiment analysis on the customer's comment
   → If (outside the 7-day refund window?)
       → If NOT outside window: request passes through for standard handling
       → If outside window, branches by customer profile:
           ├─ Standard customer (spend < 3000, sentiment ≠ very negative)
           │     → Automated email explaining the refund window has passed
           ├─ VIP customer (spend ≥ 3000)
           │     → Personalized email + Telegram alert to the team for manual review
           └─ At-risk customer (sentiment = very negative)
                 → Personalized email + urgent Telegram alert escalated to the team
```

![Workflow execution demo](workflow-demo.gif)

### Customer classification logic

| Profile | Condition | Action |
|---|---|---|
| **Standard** | Total spend < 3000, sentiment not "very negative" | Automated rejection email |
| **VIP** | Total spend ≥ 3000 | Personalized email + Telegram notification for human review |
| **At-risk** | Sentiment classified as "very negative" | Personalized email + urgent Telegram escalation |

The AI Agent returns a structured JSON output (via a Structured Output Parser) containing the customer's name, email, product, comment, sentiment classification, and purchase data pulled from the spreadsheet — which downstream nodes use to personalize each message.

## Stack

- **n8n** (self-hosted) — workflow orchestration
- **Google Gemini 2.5 Flash** — AI Agent reasoning and sentiment classification
- **Google Sheets** — customer database, queried as a tool by the AI Agent
- **Gmail** — automated customer email responses
- **Telegram** — internal team alerts for cases requiring human review

## Setup

1. Import `refund_request` into your n8n instance.
2. Configure credentials for: Google Gemini (or another supported chat model), Google Sheets OAuth2, Gmail OAuth2, and Telegram Bot API.
3. Replace the placeholders in the workflow with your own values:
   - `YOUR_SPREADSHEET_ID` → your Google Sheets document ID
   - `YOUR_TELEGRAM_CHAT_ID` → the chat/group ID that should receive alerts
   - `your-webhook-path-here` → a custom path for the Webhook node (or leave as auto-generated)
4. Your Google Sheet should contain at least these columns: `nome_cliente`, `total_gasto_cliente`, `data_ultima_compra`, and an email column used as the lookup key.
5. Connect the Webhook node to your refund request form (e.g., a Google Form via Apps Script, Typeform, or any service that can POST JSON).

## Known limitations

- **No error handling for external calls.** If the Google Sheets lookup fails, times out, or returns no match, or if the Gemini API is rate-limited/unavailable, the workflow currently has no retry logic or fallback path — the execution will simply fail.
- **No handling for "customer not found."** The prompt asks the AI Agent to return `cliente_encontrado: false` when the customer isn't in the spreadsheet, but no downstream branch currently acts on that flag — it's captured but not yet used to route the request differently.
- **Thresholds are static and untested at scale.** The 7-day window and 3000 VIP threshold were validated manually with a handful of test submissions, not with a large or adversarial dataset (e.g., ambiguous sentiment, missing fields, malformed dates).
- **No logging/observability.** There's no persisted record of which path a request took or why — useful for debugging and for auditing refund decisions, but not implemented here.

## Notes

- The 7-day refund window and the 3000 spend threshold for VIP classification are hardcoded in this version — adjust the `If` node conditions to match your own business rules.
- This project was built as part of an n8n automation course. All business logic, prompt design, and the classification model were adapted and configured for this specific use case.
- The workflow JSON in this repository has been sanitized — spreadsheet ID, Telegram chat ID, and webhook path have been replaced with placeholders.

## Author

Raul Sordi
