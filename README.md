# WooCommerce Missing Address Handler - n8n Workflow

Production-grade n8n workflow for automating missing shipping address handling from 3PL warehouses.

## Workflow Architecture (17 Nodes)

```
Webhook Trigger → Data Sanitization → Claude AI Extraction → Order Validation
                                                                ↓
┌──────────────────────────────────────────────┐
│ WooCommerce API Lookup → Address Analysis → Decision    │
└──────────────────────────────────────────────┘
                                                   ↓
        Email + SMS + Slack → Google Sheets Logging
```

## Key Features

- **Claude AI Extraction**: Intelligently parses unstructured 3PL emails
- **WooCommerce API**: Validates orders and analyzes address completeness
- **Multi-Channel Notifications**: Email, SMS (Twilio), and Slack alerts
- **Audit Trail**: Google Sheets logging with follow-up dates
- **Error Handling**: Critical error alerts, false positive detection

## Setup

1. Import `workflow.json` into your n8n instance
2. Configure credentials: Claude API, WooCommerce API, Google Sheets
3. Set environment variables
4. Activate the workflow

## Test

```bash
curl -X POST https://your-n8n/webhook/woocommerce-missing-address \
  -H "Content-Type: application/json" \
  -d '{"body":"Order #12345 missing apt number","subject":"Missing Address"}'
```

## Technology

n8n | Claude AI | WooCommerce REST API | Twilio | Slack | Google Sheets
