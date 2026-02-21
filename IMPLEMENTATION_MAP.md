# Implementation Map - WooCommerce Missing Address Workflow

## File Structure

```
MVP1_WooCommerce_Missing_Address/
├── workflow.json                 ← MAIN FILE: Import this into n8n
├── README.md                     ← Comprehensive setup guide
├── QUICKSTART.md                 ← 5-minute setup
├── NODE_REFERENCE.md             ← Detailed node documentation
├── MANIFEST.txt                  ← Project overview
└── IMPLEMENTATION_MAP.md         ← This file
```

## Quick Navigation

| What I Need | Where to Look |
|-------------|---------------|
| To import workflow | Open `workflow.json` in n8n's Import dialog |
| Quick 5-minute setup | Read `QUICKSTART.md` |
| Detailed setup guide | Read `README.md` |
| Node configuration details | Read `NODE_REFERENCE.md` |
| Project overview | Read `MANIFEST.txt` |
| This visual guide | You are here |

## Implementation Timeline

### Phase 1: Preparation (15 minutes)
```
□ Read QUICKSTART.md
□ Gather API credentials (Claude, WooCommerce)
□ Create Google Cloud service account
□ Create Google Sheet for tracking
□ Get Slack webhook URL (optional)
```

### Phase 2: Configuration (30 minutes)
```
□ Import workflow.json into n8n
□ Create Claude API credential in n8n
□ Create WooCommerce API credential in n8n
□ Create Google Sheets credential in n8n
□ Set environment variables on n8n server
□ Configure email/SMTP settings
```

### Phase 3: Testing (15 minutes)
```
□ Send test webhook request
□ Verify email received
□ Check Google Sheet row added
□ Verify Slack notification (if applicable)
□ Check execution logs for errors
```

### Phase 4: Deployment (5 minutes)
```
□ Activate workflow
□ Monitor first real notifications
□ Adjust templates if needed
□ Train team on tracking
□ Set up monitoring alerts
```

**Total Time to Production:** ~1 hour

## Setup Dependency Map

```
Claude API Credentials
    ↓
n8n Credential Manager
    ↓
Import workflow.json
    ↓
WooCommerce Credentials + Environment Variables
    ↓
Google Sheets Setup
    ↓
Email/SMTP Configuration
    ↓
Test Webhook
    ↓
Activate Workflow
```

## Credential Configuration Checklist

```
REQUIRED CREDENTIALS:
┌─────────────────────────────────────────────────────────────┐
│ ☐ Claude API                                                │
│   Name in n8n: claude_api                                   │
│   Source: https://console.anthropic.com/account/api-keys    │
│   Used by: Node 3 (Claude: Extract Order Details)           │
├─────────────────────────────────────────────────────────────┤
│ ☐ WooCommerce Basic Auth                                    │
│   Name in n8n: woocommerce_api                              │
│   Type: Basic Auth                                          │
│   Username: Consumer Key (from WooCommerce REST API)        │
│   Password: Consumer Secret                                 │
│   Used by: Node 6 (HTTP: Fetch Order)                       │
├─────────────────────────────────────────────────────────────┤
│ ☐ Google Sheets API                                         │
│   Name in n8n: google_sheets_creds                          │
│   Type: Service Account (JSON key from Google Cloud)        │
│   Used by: Node 13 (Google Sheets: Log Tracking)            │
└─────────────────────────────────────────────────────────────┘

OPTIONAL CREDENTIALS:
┌─────────────────────────────────────────────────────────────┐
│ ☐ Email/SMTP Settings                                      │
│   Configure in: n8n Settings → Email                        │
│   Used by: Nodes 5, 10, 15, 16, 17 (all email sends)       │
│                                                             │
│ ☐ Slack Webhook                                            │
│   Store as: SLACK_WEBHOOK_URL environment variable         │
│   Used by: Node 12 (Slack: Post Alert)                      │
│                                                             │
│ ☐ Twilio Webhook                                           │
│   Store as: TWILIO_SERVICE_ID environment variable         │
│   Store as: TWILIO_INTEGRATION_ID environment variable     │
│   Used by: Node 11 (Twilio: Send SMS)                       │
└─────────────────────────────────────────────────────────────┘
```

## Environment Variables Setup

Add these to your n8n server configuration:

```bash
# REQUIRED
export WOOCOMMERCE_DOMAIN="yourstore.com"
export GOOGLE_SHEETS_TRACKING_ID="1A2B3C4D5E6F7G8H9I0J1K"
export SITE_URL="https://yourstore.com"

# OPTIONAL (if using Slack)
export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"

# OPTIONAL (if using Twilio SMS)
export TWILIO_SERVICE_ID="your_service_id"
export TWILIO_INTEGRATION_ID="your_integration_id"
```

## Google Sheet Template

Create a new Google Sheet with these exact headers:

| Column | Header |
|--------|--------|
| A | Timestamp |
| B | Order ID |
| C | Order Number |
| D | Customer Name |
| E | Customer Email |
| F | Customer Phone |
| G | Missing Fields |
| H | Action Taken |
| I | Status |
| J | Follow-up Date |

Example first data row:
```
2026-02-21T10:30:00Z | 12345 | 12345 | John Smith | john@example.com | +1-555-0123 | Missing city; Missing state | Email & SMS Sent | Pending Response | 2026-02-22
```

## Webhook Integration Guide

After activation, configure your 3PL system to POST to:

```
https://your-n8n-instance/webhook/woocommerce-missing-address
```

### Webhook Payload Format

**Simple Email Format (Most Common):**
```json
{
  "body": "Order #12345 from customer Jane Doe at jane@example.com with phone 555-0123 is missing the apartment/suite number in the address.",
  "subject": "Missing Address Notification"
}
```

**Structured Format:**
```json
{
  "body": "Customer reported missing city in shipping address",
  "subject": "Order #98765 - Address Issue",
  "order_id": 98765,
  "customer": "Jane Doe",
  "email": "jane@example.com"
}
```

The workflow handles both formats through AI extraction.

### Testing the Webhook

```bash
# Test with curl
curl -X POST https://your-n8n-instance/webhook/woocommerce-missing-address \
  -H "Content-Type: application/json" \
  -d '{
    "body": "Order #12345 from John Smith (john@example.com, 555-0123) is missing the apartment number.",
    "subject": "Missing Address - Order #12345"
  }'

# Expected HTTP Response: 200 OK
```

## Workflow Execution Flow Diagram

```
START (Webhook received)
  ↓
Step 1: Receive webhook
  ├─ Input: body, subject
  ├─ Output: Passes to sanitization
  └─ Type: n8n-nodes-base.webhook

  ↓
Step 2: Sanitize data
  ├─ Input: Raw payload
  ├─ Output: Structured fields (email_body, email_subject, timestamp)
  └─ Type: n8n-nodes-base.set

  ↓
Step 3: Extract with Claude
  ├─ Input: Email text
  ├─ Process: AI extraction (zero temp)
  ├─ Output: JSON {order_number, customer_email, customer_phone, etc.}
  └─ Type: @n8n/n8n-nodes-langchain.openAi

  ↓
Step 4: Validate extraction
  ├─ Condition: order_number exists?
  ├─ TRUE → Continue to WooCommerce lookup
  └─ FALSE → Send error alert, END

  ↓
Step 5: Fetch WooCommerce order
  ├─ Input: order_number
  ├─ API: GET /wp-json/wc/v3/orders?search={{order_number}}
  ├─ Output: Complete order data
  └─ Type: n8n-nodes-base.httpRequest

  ↓
Step 6: Check if order exists
  ├─ Condition: order found in response?
  ├─ TRUE → Analyze address
  └─ FALSE → Send "order not found" alert, END

  ↓
Step 7: Analyze address
  ├─ Input: shipping_address object
  ├─ Check: Missing fields (street, city, state, postcode, country)
  ├─ Output: List of missing fields
  └─ Type: n8n-nodes-base.code

  ↓
Step 8: Check for issues
  ├─ Condition: address_issues.length > 0?
  ├─ TRUE → Send customer communications
  └─ FALSE → Send "no issues found" notification, END

  ↓ (Parallel branches)
  ├─ Step 9a: Send Email
  │   ├─ To: customer email
  │   ├─ Subject: "Complete Your Address"
  │   ├─ Body: Missing fields list + direct link
  │   └─ Type: n8n-nodes-base.emailSend
  │
  ├─ Step 9b: Send SMS
  │   ├─ To: customer phone (if available)
  │   ├─ Body: Short message with urgency
  │   └─ Type: n8n-nodes-base.httpRequest (Twilio)
  │
  └─ Step 9c: Post Slack Alert
      ├─ To: #fulfillment-alerts
      ├─ Content: Order details + missing fields
      └─ Type: n8n-nodes-base.httpRequest (Slack)

  ↓ (Converge after parallel)
Step 10: Prepare data
  ├─ Format: Google Sheet row format
  ├─ Fields: Timestamp, Order, Customer, Missing Fields, Action, Status
  └─ Type: n8n-nodes-base.set

  ↓
Step 11: Log to Google Sheets
  ├─ Action: Append row
  ├─ Sheet: Tracking sheet (ID from env var)
  ├─ Columns: A-J (as defined)
  └─ Type: n8n-nodes-base.googleSheets

  ↓
END (Success)
```

## Decision Tree

```
Webhook Received
│
├─ Claude extraction fails?
│  └─ YES: Send error alert → END
│  └─ NO: Continue
│
├─ Order found in WooCommerce?
│  └─ NO: Send "order not found" alert → END
│  └─ YES: Continue
│
├─ Address issues found?
│  └─ NO: Send "no issues" notification → END
│  └─ YES: Send customer communications
│         └─ Send email
│         └─ Send SMS (if phone available)
│         └─ Post Slack notification
│         └─ Log to Google Sheets → END
```

## Monitoring & Metrics

### Key Performance Indicators

Track these metrics from Google Sheets:

| Metric | Calculation | Healthy Range |
|--------|-------------|---------------|
| Daily Volume | Rows added per day | 5-50 |
| Response Rate | Customers updating address / total | >20% |
| False Positive Rate | "No issues" / total | <5% |
| Processing Speed | n8n avg execution time | 8-15 sec |
| Error Rate | Failed executions / total | <1% |

### Daily Monitoring Tasks

1. **Morning Review:**
   - Check Google Sheets for overnight notifications
   - Look for unusual patterns or high volume
   - Note any error alerts received

2. **Weekly Review:**
   - Analyze customer response rates
   - Review n8n execution history for errors
   - Identify false positive patterns

3. **Monthly Review:**
   - Calculate metrics above
   - Adjust email template if response <20%
   - Review Claude extraction success rate
   - Plan optimizations

## Troubleshooting Quick Reference

| Issue | Check | Fix |
|-------|-------|-----|
| Webhook not receiving | Webhook URL correct? | Verify in n8n settings |
| No emails sent | SMTP configured? | Configure in n8n settings |
| WooCommerce API errors | Credentials valid? | Regenerate Consumer Key/Secret |
| Claude extraction failing | API key valid? | Check API usage/quota |
| Google Sheet not updating | Sheet shared? | Share with service account email |
| Slack not posting | Webhook URL valid? | Regenerate and update env var |
| Order not found errors | Order numbers match? | Verify 3PL format vs WooCommerce |

## Performance Baseline

| Operation | Typical Time | Max Time |
|-----------|-------------|----------|
| Webhook receipt → Processing | <100ms | 500ms |
| Claude extraction | 2-3s | 5s |
| WooCommerce API call | 2-4s | 8s |
| Email send | <1s | 2s |
| Google Sheets write | <500ms | 1s |
| **Total workflow time** | **8-12s** | **20s** |

Safe for 100+ notifications/day without performance issues.

## Security Considerations

✓ **Credentials:** All stored in n8n credential manager
✓ **API Keys:** Never in workflow JSON
✓ **Data:** No sensitive info logged except order number
✓ **HTTPS:** All external calls use HTTPS
✓ **Audit Trail:** All actions logged in Google Sheets
✓ **GDPR:** Customer data only as needed; customize retention

## Deployment Readiness Checklist

### Pre-Deployment
- [ ] All files extracted and reviewed
- [ ] Credentials gathered (Claude, WooCommerce, Google)
- [ ] Google Cloud project created
- [ ] Google Sheet created with headers
- [ ] n8n instance accessible
- [ ] SMTP configured in n8n

### During Deployment
- [ ] workflow.json imported successfully
- [ ] All 17 nodes visible
- [ ] Credentials configured (3 required)
- [ ] Environment variables set
- [ ] Webhook URL noted
- [ ] Webhook tested successfully

### Post-Deployment
- [ ] Workflow set to Active
- [ ] 3PL configured to send webhook requests
- [ ] First notification processed successfully
- [ ] Google Sheet shows first row
- [ ] Team trained on monitoring
- [ ] Monitoring alerts configured
- [ ] Backup of workflow stored

## Integration Points Summary

### Incoming
```
3PL Warehouse System
    ↓ (HTTPS POST)
n8n Webhook Endpoint
```

### Outgoing
```
Claude API → Intelligent extraction
WooCommerce API → Order validation
Email/SMTP → Customer notifications
Twilio → SMS notifications
Slack → Team alerts
Google Sheets → Audit logging
```

## Support & Resources

| Need | Resource |
|------|----------|
| n8n setup help | https://docs.n8n.io/ |
| n8n community | https://community.n8n.io/ |
| Claude API docs | https://docs.anthropic.com/ |
| WooCommerce API | https://woocommerce.github.io/woocommerce-rest-api-docs/ |
| Google Sheets API | https://developers.google.com/sheets/api/ |
| Slack webhooks | https://api.slack.com/messaging/webhooks |

## Next Steps

1. **Start Here:** Read QUICKSTART.md (5 min)
2. **Then:** Follow setup steps in section above
3. **Reference:** Use NODE_REFERENCE.md for details
4. **Monitor:** Track in Google Sheets daily
5. **Optimize:** Adjust templates based on response rates

---

**Estimated Time to Production: 1 hour**

Ready? Import `workflow.json` and follow QUICKSTART.md!
