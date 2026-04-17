# Email System Setup Guide

## What This Workflow Does

Two independent email systems in one n8n workflow:

**1. Campaign Emails** — Reads drafts from Google Sheets, sends to all subscribers on a schedule (with open/click tracking + unsubscribe).

**2. Transactional Emails (Webhook)** — Your webapp POSTs `{operation, payload}` to a webhook. The workflow fetches the matching EJS template from Postgres (`fine_templates` table), renders it with the payload data, and sends it via Gmail.

---

## Files

| File | Purpose |
|------|---------|
| `newsletter-workflow.json` | Import into n8n |
| `002_create_fine_templates.sql` | Run against your Postgres DB (creates `fine_templates` table + seeds `pay_later` template) |

---

## Setup

### 1. Run the Migration

```bash
psql -U your_user -d your_db -f 002_create_fine_templates.sql
```

This creates the `fine_templates` table and seeds it with a `pay_later` EJS email template.

### 2. Import the Workflow

n8n → Workflows → Import from File → select `newsletter-workflow.json`

### 3. Connect Credentials

Open each node and attach the correct credential:

- **All Google Sheets nodes** → your Google Sheets OAuth2 credential
- **All Gmail nodes** (3 total) → your Gmail OAuth2 credential
- **"Fetch Template from Postgres"** → your Postgres credential

### 4. Link Your Google Sheet

For the campaign side, select your spreadsheet + sheet tab in every Google Sheets node. Required sheets: `Emails`, `Subscribers`, `Tracking`.

### 5. Set Webhook Base URL

In both campaign JavaScript nodes, replace:
```
YOUR_N8N_WEBHOOK_URL
```
with your actual n8n webhook base URL (e.g. `https://your-n8n.example.com/webhook`).

### 6. Install EJS in n8n

The "Render EJS Template" node uses `require('ejs')`. Make sure EJS is available:

```bash
cd /path/to/n8n
npm install ejs
```

Or if using Docker, add it to your n8n image or use `NODE_FUNCTION_ALLOW_EXTERNAL=ejs` environment variable.

### 7. Activate

Toggle the workflow to **Active**.

---

## Transactional Email — How to Call It

Your webapp sends a POST to:

```
POST https://your-n8n.example.com/webhook/send-email
Content-Type: application/json
```

**Payload structure:**

```json
{
  "operation": "pay_later",
  "payload": {
    "to": "driver@example.com",
    "pcn_number": "PCN-2024-00123",
    "vehicle_registration": "AB12 CDE",
    "operator_name": "ParkingEye",
    "contravention_date": "15 January 2026",
    "full_penalty_amount": "60.00",
    "payment_deadline": "28 February 2026",
    "portal_url": "https://portal.parkingeye.co.uk"
  }
}
```

**Flow:**
```
Webhook → IF (operation = "pay_later") → Fetch EJS template from Postgres → Render with payload → Gmail send → 200 OK
                                      → else → 400 "Unknown operation"
```

**Response (success):**
```json
{ "success": true, "message": "Email sent", "to": "driver@example.com" }
```

The EJS template uses standard `<%= variable %>` syntax. Every key in `payload` becomes available in the template. The seeded `pay_later` template expects: `to`, `pcn_number`, `vehicle_registration`, `operator_name`, `contravention_date`, `full_penalty_amount`, and optionally `payment_deadline` and `portal_url`.

---

## Adding New Operations

1. Insert a new row into `fine_templates`:

```sql
INSERT INTO fine_templates (name, subject, template)
VALUES ('dispute_update', 'Dispute Update — PCN <%= pcn_number %>', '<html>...your EJS...</html>');
```

2. Add a new branch in the **Check Operation** IF node (or convert it to a Switch node for multiple operations).

3. Call the webhook with `"operation": "dispute_update"`.

---

## Campaign Side (Unchanged)

- **Schedule Trigger** sends the next `Draft` email from Google Sheets to all `Subscribed` addresses.
- **Manual trigger** (top row) does the same thing for testing.
- **Tracking webhooks** log opens and clicks to the `Tracking` sheet.
- **Unsubscribe webhook** marks subscribers as unsubscribed.

Refer to the Google Sheets column layouts from the original setup if needed:
- `Emails` sheet: `Email ID`, `Subject`, `HTML Body`, `Status`, `Sent Date`
- `Subscribers` sheet: `Email`, `Name`, `Status`, `Subscribed Date`, `Unsubscribed Date`
- `Tracking` sheet: `Tracking ID`, `Email ID`, `Subscriber Email`, `Event Type`, `Clicked URL`, `Timestamp`