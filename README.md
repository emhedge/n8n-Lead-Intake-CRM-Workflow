# n8n Lead Intake CRM Workflow

A production-ready lead intake automation built with n8n. Accepts form submissions via webhook, validates and deduplicates incoming leads, enriches them with company data, stores them to Google Sheets, and fires multi-channel notifications — all with a proper error handling workflow.

Built as a portfolio project to demonstrate real-world n8n workflow automation skills.

---

## What it does

1. **Receives** a lead submission via HTTP POST webhook from an HTML form
2. **Validates** that required fields (email, company) are present and non-empty
3. **Deduplicates** against existing leads in Google Sheets using name + email as the dedupe key
4. **Enriches** the lead with company data (industry, headcount, country, LinkedIn) via Abstract API
5. **Stores** the enriched lead record to Google Sheets
6. **Notifies** the team via Telegram
7. **Confirms** receipt to the lead via email
8. **Handles errors** gracefully with a dedicated error workflow that alerts via Telegram

---

## Architecture

```
Lead Webhook
    │
    ▼
Map & Timestamp Fields
    │
    ▼
Validate Required Fields (IF)
    ├── false → Respond: Validation Error (400)
    └── true
          │
          ▼
    Lookup Existing Leads (Google Sheets)
          │
          ▼
    Check Duplicate (Code)
          │
          ▼
    Is Duplicate? (IF)
          ├── true → Mark as Duplicate → Respond: Duplicate (200)
          └── false
                │
                ▼
          Enrich Company Data (Abstract API)
                │
                ▼
          Merge Lead & Enrichment Data (Code)
                │
                ▼
          Store Lead in Sheet (Google Sheets)
                │
                ▼
          Notify Team (Telegram)
                │
                ▼
          Send an Email (Gmail SMTP)
                │
                ▼
          Respond: Success (200)
```

**Error Workflow** (separate): Error Trigger → Notify Error (Telegram)

---

## Tech stack

| Tool | Purpose |
|---|---|
| [n8n](https://n8n.io) | Workflow automation engine |
| [ngrok](https://ngrok.com) | Expose local webhook to the internet |
| [Google Sheets](https://sheets.google.com) | Lead storage database |
| [Abstract API – Company Enrichment](https://www.abstractapi.com/api/company-enrichment-api) | Domain-based company data enrichment |
| [Telegram Bot API](https://core.telegram.org/bots/api) | Team notifications |
| Gmail SMTP | Lead confirmation emails |
| HTML/CSS/JS | Lead intake form frontend |
| Docker | n8n containerized deployment |

---

## Data schema

| Field | Type | Source |
|---|---|---|
| `lead_id` | string | Auto-generated (timestamp) |
| `name` | string | Form submission |
| `email` | string | Form submission — dedupe key |
| `company` | string | Form submission — dedupe key |
| `message` | string | Form submission |
| `source` | string | Hardcoded: `webhook-form` |
| `submitted_at` | ISO timestamp | Generated at webhook trigger time |
| `status` | string | Set by workflow: `new` or `duplicate` |
| `enriched_industry` | string | Abstract API |
| `enriched_employees` | number | Abstract API |
| `enriched_country` | string | Abstract API |
| `enriched_linkedin` | string | Abstract API |

---

## Prerequisites

- [n8n](https://n8n.io) running locally via Docker (port 5678)
- [ngrok](https://ngrok.com) account with a static domain
- Google account with Sheets API and Drive API enabled
- Abstract API account (free tier — Company Enrichment product)
- Telegram bot token and chat ID
- Gmail account with an App Password generated

---

## Setup

### 1. Start n8n via Docker

```bash
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -e GENERIC_TIMEZONE=Asia/Taipei \
  -e TZ=Asia/Taipei \
  -e N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true \
  -e N8N_RUNNERS_ENABLED=true \
  -e N8N_HOST=localhost \
  -e N8N_PORT=5678 \
  -e N8N_PROTOCOL=http \
  -e WEBHOOK_URL=http://localhost:5678 \
  -v "/path/to/your/n8n/data:/home/node/.n8n" \
  n8nio/n8n:latest
```

### 2. Start ngrok

```bash
ngrok http 5678
```

Note your ngrok URL — update `WEBHOOK_URL` in `lead-intake-form.html` to match.

### 3. Configure secrets

Copy the example config file and fill in your values:

```bash
cp config.example.js config.js
```

Edit `config.js` with your actual webhook URL and secret token:

```javascript
window.CONFIG = {
  WEBHOOK_URL: "https://your-ngrok-url/webhook/new-lead",
  WEBHOOK_SECRET: "your-secret-token"
};
```

> ⚠️ `config.js` is gitignored and should never be committed. All real secrets stay local.

### 4. Configure credentials in n8n

| Credential | Type | Notes |
|---|---|---|
| Google Sheets | OAuth2 | Requires Sheets API + Drive API enabled in Google Cloud Console |
| Abstract API | HTTP Header (query param) | Pass `api_key` as a query parameter |
| Telegram | Telegram Bot API | Bot token from @BotFather |
| Gmail | SMTP | Host: `smtp.gmail.com`, Port: `465`, use App Password |

### 5. Import the workflow

1. Open n8n at `http://localhost:5678`
2. Go to **Workflows → Import**
3. Upload `lead-intake-workflow.json`
4. Update all credentials in each node
5. Activate the workflow

### 6. Serve the form

```bash
cd /path/to/form
python3 -m http.server 8000
```

Open `http://localhost:8000/lead-intake-form.html`

---

## Testing

**Valid new lead:**
```bash
curl -X POST https://<your-ngrok-url>/webhook/new-lead \
  -H "Content-Type: application/json" \
  -d '{"name":"Jane Smith","email":"jane@acme.com","company":"Acme Corp","message":"Hello"}'
```
Expected: 200, row appended to sheet, Telegram notification, confirmation email sent.

**Duplicate lead:**
```bash
curl -X POST https://<your-ngrok-url>/webhook/new-lead \
  -H "Content-Type: application/json" \
  -d '{"name":"Jane Smith","email":"jane@acme.com","company":"Acme Corp","message":"Again"}'
```
Expected: 200 with `{"status":"duplicate"}`, no new row written.

**Missing required fields:**
```bash
curl -X POST https://<your-ngrok-url>/webhook/new-lead \
  -H "Content-Type: application/json" \
  -d '{"name":"No Company","email":"test@test.com","company":"","message":""}'
```
Expected: 400 with `{"status":"error"}`.

---

## n8n skills demonstrated

- Webhook trigger configuration (path, method, response mode)
- Data mapping and transformation (Edit Fields / Set nodes)
- Conditional branching (IF nodes)
- Deduplication logic against a live data source
- External API enrichment via HTTP Request node
- Data merging across nodes using `$('Node Name').first().json`
- Google Sheets read and write operations
- Multi-channel notifications (Telegram + email)
- Proper error workflow using Error Trigger node
- Node naming and canvas organization best practices

---

## Author

Evyn Hedgpeth
[linkedin.com/in/evynhedgpeth](https://linkedin.com/in/evynhedgpeth) · [medium.com/@emhedge](https://medium.com/@emhedge)
