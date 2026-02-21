# Quick Start Guide - WooCommerce Missing Address Handler

## 30-Second Overview

This n8n workflow automatically handles customer address issues reported by 3PL warehouses. It extracts order details from emails using AI, validates them against WooCommerce, and sends smart customer notifications.

## 5-Minute Setup

### 1. Import Workflow
- Open n8n → Workflows → Import from File
- Select `workflow.json`

### 2. Add Credentials (5 services)
In n8n Settings → Credentials, create:
- **Claude API**: Get key from https://console.anthropic.com/
- **WooCommerce Basic Auth**: Key/Secret from your WordPress REST API settings
- **Google Sheets**: Service account JSON from Google Cloud Console
- **Email**: Configure your SMTP settings (Settings → Email)
- **Slack Webhook** (optional): From your Slack workspace

### 3. Set Environment Variables
Add these to your n8n server config:
```
WOOCOMMERCE_DOMAIN=yourstore.com
GOOGLE_SHEETS_TRACKING_ID=YOUR_SHEET_ID (from Google Sheet URL)
SLACK_WEBHOOK_URL=https://hooks.slack.com/...
SITE_URL=https://yourstore.com
```

### 4. Create Google Sheet
1. Create a new Google Sheet
2. Add headers: Timestamp | Order ID | Order Number | Customer Name | Customer Email | Customer Phone | Missing Fields | Action Taken | Status | Follow-up Date
3. Share with the Google service account email
4. Copy the Sheet ID from the URL and save to `GOOGLE_SHEETS_TRACKING_ID`

### 5. Test It
Send a webhook POST:
```bash
curl -X POST https://your-n8n.com/webhook/woocommerce-missing-address \
  -H "Content-Type: application/json" \
  -d '{
    "body": "Order #12345 from Jane Doe (jane@example.com, 555-0123) is missing the apartment number in the address.",
    "subject": "Missing Address - Order 12345"
  }'
```

Expected result:
- Email sent to jane@example.com
- Row added to tracking sheet
- Slack notification posted (if configured)

## Workflow Execution Flow

```
3PL Webhook
    ↓
Extract Details (Claude AI)
    ↓
Validate Order Number → [Error? → Alert team]
    ↓
Fetch from WooCommerce
    ↓
Check if Order Exists → [Not found? → Alert team]
    ↓
Analyze Address Fields
    ↓
Missing Fields?
    ├─ YES → Send Email + SMS + Slack Alert
    └─ NO → Log as false positive
    ↓
Log to Google Sheet
    ↓
Done
```

## Key Files in This Package

| File | Purpose |
|------|---------|
| `workflow.json` | Complete n8n workflow - import this into n8n |
| `README.md` | Comprehensive setup and configuration guide |
| `QUICKSTART.md` | This file - quick reference |

## Node Reference (What Each Does)

| Node Name | Type | Purpose |
|-----------|------|---------|
| Webhook: 3PL Notification | Webhook | Receives incoming alerts from warehouse |
| Sanitize Input Data | Set | Extracts and validates email content |
| Claude: Extract Order Details | OpenAI/Claude | Uses AI to parse unstructured email into structured data |
| Check: Order Number Valid | If | Routes if extraction succeeded |
| HTTP: Fetch Order from WooCommerce | HTTP | Gets order details from your store |
| Check: Order Found | If | Routes if order exists |
| Code: Analyze Address Completeness | Code | Identifies missing address fields |
| Check: Address Issues Found | If | Routes based on address completeness |
| Email: Request Address Correction | Email | Sends customer the address correction request |
| Twilio: Send SMS Reminder | HTTP | Sends optional SMS (via Twilio) |
| Slack: Post Alert to Fulfillment | HTTP | Notifies your team on Slack |
| Google Sheets: Log Tracking | Google Sheets | Records action in tracking spreadsheet |

## Configuration Checklist

- [ ] Claude API credentials added
- [ ] WooCommerce API credentials added
- [ ] Google Sheets API credentials added
- [ ] Email/SMTP configured
- [ ] Slack webhook added (optional)
- [ ] Environment variables set on server
- [ ] Google Sheet created and shared
- [ ] Webhook tested successfully
- [ ] Workflow set to Active

## Common Webhook Formats Your 3PL Might Send

### Format 1: Email-like
```json
{
  "body": "Order #12345 from John Doe has incomplete address info.",
  "subject": "Missing Address Alert"
}
```

### Format 2: Structured
```json
{
  "order_number": "12345",
  "customer_email": "john@example.com",
  "issue": "Missing city in address"
}
```

The Claude AI is flexible and can extract from both formats.

## Monitoring

### Daily Check
Look at your tracking Google Sheet to see:
- How many missing address alerts came in
- Which orders were resolved
- Any anomalies or false positives

### Weekly Check
Review n8n execution logs (Workflows → Executions) for:
- Any errors or failed runs
- Average execution time
- Response codes from WooCommerce API

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Order not found" errors | Check if order numbers in warehouse emails match WooCommerce format |
| Email not sending | Verify SMTP settings in n8n Email configuration |
| Google Sheet not updating | Check sheet sharing; verify service account has edit access |
| Claude extraction not working | Review warehouse email format; update extraction prompt if needed |
| Slack notifications failing | Regenerate webhook URL; check channel name is correct |

## Support Resources

- **n8n Docs:** https://docs.n8n.io/
- **n8n Community:** https://community.n8n.io/
- **Claude API Docs:** https://docs.anthropic.com/
- **WooCommerce REST API:** https://woocommerce.github.io/woocommerce-rest-api-docs/

## Next Steps (After Setup Works)

1. **Analyze Results:** Run for 1-2 weeks, review tracking sheet
2. **Optimize:** Adjust email templates based on customer response rates
3. **Extend:** Add retry logic if customers don't respond within 24 hours
4. **Integrate:** Connect to your order management system or ERP

## API Rate Limits & Performance

- **Execution Time:** 8-12 seconds per notification
- **WooCommerce:** Default 10 requests/second (usually not a bottleneck)
- **Claude:** Standard API tier limits (~1.5M tokens/min)
- **Google Sheets:** 60 requests/minute (one per execution, plenty of headroom)

Safe to run 100+ notifications per day without issues.

---

**Ready to go?** Import the `workflow.json` file and follow the setup steps above. You'll be live in under 10 minutes.
