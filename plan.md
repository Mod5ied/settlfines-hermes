Here's your complete workflow and setup guide. The JSON contains 6 interconnected flows that replicate your canvas:
Main canvas (what you see in the image):

Bottom row — Schedule Trigger → get email → Update rows → get subscriber list → JavaScript (inject tracking) → set fields → Gmail send
Top row — Identical pipeline triggered by manual "Execute" button for testing

Supporting flows (for the native tracking system):

Open tracking — Webhook serves a 1×1 pixel, logs opens to Google Sheets
Click tracking — Webhook logs the click, then redirects to the real URL
Unsubscribe — Updates subscriber status in Sheets, shows a confirmation page
Subscribe — POST endpoint to add new subscribers with validation

To get running, you'll need to:

Import the JSON into n8n
Create a Google Sheet with three tabs (Emails, Subscribers, Tracking) — column layouts are in the guide
Connect your Google Sheets OAuth2 and Gmail OAuth2 credentials to each node
Replace YOUR_N8N_WEBHOOK_URL in both JavaScript nodes with your actual n8n webhook base URL
Activate the workflow

The setup guide covers everything in detail including the sheet structure, testing steps, and troubleshooting.