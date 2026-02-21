# WooCommerce Missing Address Handler - 3PL Integration Workflow

## Overview

This is a production-grade n8n workflow designed to automate the handling of incomplete shipping addresses reported by 3PL (Third Party Logistics) warehouses. The workflow monitors incoming notifications, validates order data against WooCommerce, and automatically sends intelligent customer communications to resolve address issues before shipment.

**Workflow Status:** Ready for Production Deployment
**Version:** 1.0.0
**Last Updated:** February 2026

## Problem Statement

When 3PL warehouses attempt to fulfill orders, they sometimes encounter incomplete or missing shipping addresses. Manual handling of these cases is:
- Time-consuming and error-prone
- Creates shipping delays
- Results in customer dissatisfaction
- Lacks visibility and tracking

This workflow automates the entire process from notification to resolution.

## Workflow Architecture

### Step 1: Webhook Trigger
**Node:** `Webhook: 3PL Notification`

Receives incoming HTTP POST requests from your 3PL warehouse system when a missing address is detected.

**Expected Payload Format:**
```json
{
  "body": "Order #98765 from customer John Smith is missing apartment/unit number in address.",
  "subject": "Missing Address Notification - Order 98765"
}
```

### Step 2: Data Sanitization
**Node:** `Sanitize Input Data`

Extracts and validates the incoming email data, preparing it for AI processing.

**Output:**
- `email_body`: Raw email content
- `email_subject`: Email subject line
- `timestamp`: Processing timestamp

### Step 3: AI Extraction (Claude)
**Node:** `Claude: Extract Order Details`

Uses Claude AI to intelligently extract structured data from unstructured warehouse emails.

**Extracted Information:**
- Order number (with fuzzy matching for various formats)
- Customer name and contact information
- Specific address issues
- Warehouse name
- Date reported

**Intelligence:** The Claude node uses zero-temperature prompts (deterministic) to reliably parse various email formats without hallucination.

### Step 4: Validation Check
**Node:** `Check: Order Number Valid`

Validates that an order number was successfully extracted. If not, sends an error alert to the support team.

**Error Path:** Routes to `Send Error Alert` node if extraction fails.

### Step 5: WooCommerce Order Lookup
**Node:** `HTTP: Fetch Order from WooCommerce`

Makes an authenticated API call to WooCommerce REST API to retrieve the complete order details using the extracted order number.

**Authentication:** Basic Auth with WooCommerce Consumer Key/Secret
**Endpoint:** `/wp-json/wc/v3/orders`
**Timeout:** 30 seconds

### Step 6: Address Analysis
**Node:** `Code: Analyze Address Completeness`

Node-JS code that inspects the order's shipping address and identifies exactly which fields are missing:
- Street address (address_1)
- City
- State/Province
- Postal code
- Country

Generates a human-readable list of missing fields for communication.

**Output:**
- `has_address_issues`: Boolean indicating if problems exist
- `address_issues`: Array of specific missing fields
- Complete order metadata for downstream nodes

### Step 7: Address Issue Decision
**Node:** `Check: Address Issues Found`

Routes the workflow based on whether address issues actually exist:
- **True Path:** Send customer communications (email, SMS, Slack notification)
- **False Path:** Log as false positive and notify support

### Step 8: Customer Communication

#### Email Notification
**Node:** `Email: Request Address Correction`

Sends an HTML-formatted email to the customer with:
- Clear explanation of missing information
- Specific list of which fields need completion
- Direct link to update address in their account
- Urgency messaging about shipment delays

**Template:** Professional HTML with company branding capability

#### SMS Reminder
**Node:** `Twilio: Send SMS Reminder`

Sends a concise SMS message (via Twilio webhook) for time-sensitive notification, useful when email might be missed.

**Content:** Short message with order number and link to address update page

#### Slack Alert
**Node:** `Slack: Post Alert to Fulfillment`

Posts a formatted notification to your fulfillment team's Slack channel with:
- Order number and customer details
- List of missing address fields
- Order status and total value
- Direct order link

**Channel:** `#fulfillment-alerts` (configurable)

### Step 9: Tracking & Logging
**Node:** `Google Sheets: Log Tracking`

Appends a row to a Google Sheet for complete audit trail and analytics:
- Timestamp of alert
- Order and customer details
- Specific missing fields
- Action taken (email/SMS sent)
- Current status
- Automated follow-up date (24 hours)

**Columns:**
- A: Timestamp
- B: Order ID
- C: Order Number
- D: Customer Name
- E: Customer Email
- F: Customer Phone
- G: Missing Fields
- H: Action Taken
- I: Status
- J: Follow-up Date

### Alternative Paths & Error Handling

#### Order Not Found
**Node:** `Send: Order Not Found Alert`

If the order number doesn't exist in WooCommerce:
- Logs the issue with the extracted order number
- Alerts support team for manual investigation
- Prevents false notifications to non-existent customers

#### Address Already Complete
**Node:** `Send: No Issues Found Notification`

If the analysis determines the address is actually complete:
- Logs as a false positive
- Notifies support team
- Suggests review of 3PL notification logic

#### Critical Error Handler
**Node:** `Send: Critical Error Alert`

Catches any workflow execution errors and:
- Sends immediate alert to ops team
- Includes execution ID for debugging
- Prevents silent failures

## Installation & Setup

### Prerequisites

1. **n8n Instance:** Self-hosted or cloud deployment
2. **Active Integrations:**
   - Claude API access (via OpenAI-compatible endpoint)
   - WooCommerce REST API credentials
   - Google Sheets API credentials
   - Twilio account (optional, for SMS)
   - Slack workspace (optional, for notifications)

### Step-by-Step Setup

#### 1. Import the Workflow

1. Open your n8n instance
2. Click "Workflows" → "Import from File"
3. Select `workflow.json`
4. Review and confirm the import

#### 2. Configure Credentials

Create credentials in n8n for each service:

**Claude API:**
```
Name: claude_api
Type: OpenAI API
API Key: sk-ant-... (your Anthropic API key)
Base URL: https://api.anthropic.com (if needed)
```

**WooCommerce API:**
```
Name: woocommerce_api
Type: Basic Auth
Username: Your Consumer Key
Password: Your Consumer Secret
```

Get these from WooCommerce: Settings → Advanced → REST API

**Google Sheets:**
```
Name: google_sheets_creds
Type: Google Service Account
JSON Key: (download from Google Cloud Console)
```

**Twilio (Optional):**
Configure as environment variable `TWILIO_SERVICE_ID` and `TWILIO_INTEGRATION_ID`

#### 3. Configure Environment Variables

In your n8n instance, set these environment variables:

```
WOOCOMMERCE_DOMAIN=yourstore.com
GOOGLE_SHEETS_TRACKING_ID=1A2B3C4D5E6F7G8H9I0J1K
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
TWILIO_SERVICE_ID=your_service_id
TWILIO_INTEGRATION_ID=your_integration_id
SITE_URL=https://yourstore.com
```

#### 4. Create Google Sheet Template

Create a Google Sheet with these headers in the first row:

| A | B | C | D | E | F | G | H | I | J |
|---|---|---|---|---|---|---|---|---|---|
| Timestamp | Order ID | Order Number | Customer Name | Customer Email | Customer Phone | Missing Fields | Action Taken | Status | Follow-up Date |

Share this sheet with your Google Service Account email (found in the JSON key).

#### 5. Set Up Slack Webhook (Optional)

1. Go to your Slack workspace settings
2. Create an Incoming Webhook for the `#fulfillment-alerts` channel
3. Copy the webhook URL to `SLACK_WEBHOOK_URL` environment variable

#### 6. Test the Workflow

**Manual Test:**

1. Click "Test" in the workflow editor
2. Send a test POST request:

```bash
curl -X POST https://your-n8n-instance/webhook/woocommerce-missing-address \
  -H "Content-Type: application/json" \
  -d '{
    "body": "We have Order #12345 from customer Jane Doe, email jane@example.com, phone 555-0123. The shipping address is missing the apartment number. Address on file: 123 Main St, New York, NY 10001.",
    "subject": "Missing Address Notification - Order 12345"
  }'
```

3. Monitor execution logs for errors
4. Verify:
   - Email received by test customer
   - Row added to Google Sheet
   - Slack notification posted

#### 7. Activate the Workflow

1. Toggle the workflow to "Active"
2. Workflow is now ready to receive webhook calls

## Integration Points

### Incoming: 3PL Warehouse System

Your 3PL provider should POST notifications to:
```
https://your-n8n-instance/webhook/woocommerce-missing-address
```

**Payload Requirements:**
- Must include email-like content with order number
- Should include customer context

### Outgoing: WooCommerce

- **API Endpoint:** `https://yourstore.com/wp-json/wc/v3/orders`
- **Authentication:** REST API credentials required
- **Operations:** Read-only (GET requests for order data)

### Outgoing: Google Sheets

- **Purpose:** Audit trail and analytics
- **Access:** Append-only via Google Sheets API
- **Frequency:** One row per processed notification

### Outgoing: Email

- **Recipients:**
  - Customers (address correction requests)
  - Support team (alerts)
- **Provider:** n8n's built-in email or SMTP
- **Configuration:** Configure in n8n settings → Email

### Outgoing: SMS (Optional)

- **Provider:** Twilio
- **Recipients:** Customers with valid phone numbers
- **Fallback:** Skipped if no phone number available

### Outgoing: Slack (Optional)

- **Channel:** #fulfillment-alerts
- **Frequency:** One message per missing address case
- **Includes:** Order details and missing fields

## Monitoring & Maintenance

### Key Metrics to Track

1. **Workflow Success Rate:** Should be >99%
2. **Average Processing Time:** Typically <15 seconds
3. **False Positive Rate:** Should be <5% (incomplete address claims that don't exist)
4. **Customer Response Rate:** Track follow-up conversions

### Logs & Debugging

1. **n8n Execution History:** View in Workflows → Executions
2. **Google Sheet Logs:** Raw data for all processed orders
3. **Error Tracking:** Check "Sent: Critical Error Alert" node for exceptions

### Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| "Order not found in WooCommerce" | Order number extraction failed | Review Claude extraction prompt; check 3PL email format consistency |
| Email not received | SMTP misconfigured | Check n8n email settings; verify recipient domain isn't blocking |
| Slack notifications failing | Invalid webhook URL | Regenerate Slack webhook; verify URL format |
| Google Sheet errors | Permission issues | Ensure service account email has edit access to sheet |
| SMS not sending | Twilio account inactive | Check Twilio balance; verify phone number format |

### Regular Maintenance Tasks

- **Weekly:** Review Google Sheet for false positives; adjust Claude extraction prompt if needed
- **Monthly:** Analyze customer response rates; optimize email templates if response is low
- **Quarterly:** Review error logs; update integration credentials before expiration

## Production Deployment Checklist

Before deploying to production:

- [ ] All credentials are stored in n8n's credential manager (not hardcoded)
- [ ] Environment variables are configured on the server
- [ ] Error handlers are configured to alert correct team email
- [ ] Google Sheet is created and shared with service account
- [ ] Slack webhook is active (if using notifications)
- [ ] Email/SMS templates are reviewed and branded
- [ ] Test webhook call is successful
- [ ] Workflow is set to "Active"
- [ ] Team is trained on monitoring Google Sheet
- [ ] On-call rotation has n8n access for troubleshooting
- [ ] Backup of workflow JSON is stored in version control

## Advanced Customization

### Extending the Workflow

**Add Database Logging:**
```
Insert a PostgreSQL node after "Prepare Google Sheet Row" to also log to your database
```

**Add Manual Review Step:**
```
Insert a "Wait for Webhook" node with timeout to allow manual address correction before sending to customer
```

**Add Address Validation API:**
```
Insert an address validation node (e.g., SmartyStreets, USPS) between "Analyze Address" and "Address Issue Check" to verify address completeness programmatically
```

**Add Retry Logic:**
```
Configure workflow to automatically resend address request after 3 days if customer doesn't respond
```

### Customizing Templates

All customer-facing messages can be customized:

1. **Email Template:** Modify the HTML in the `Email: Request Address Correction` node
2. **SMS Template:** Edit the message text in the `Twilio: Send SMS Reminder` node
3. **Slack Template:** Adjust fields in the `Slack: Post Alert to Fulfillment` node

### Claude Extraction Tuning

If warehouse emails have varying formats, refine the extraction prompt in the `Claude: Extract Order Details` node. Examples of improvements:

- Add specific order number patterns used by your warehouses
- Include company-specific abbreviations for warehouse names
- Reference specific address format variations you see

## Support & Troubleshooting

### Getting Help

1. **n8n Community:** https://community.n8n.io/
2. **Documentation:** https://docs.n8n.io/
3. **Workflow Issues:** Check execution logs in n8n UI
4. **Claude API Issues:** Check https://console.anthropic.com/account/usage

### Export & Backup

To backup this workflow:

```bash
# In n8n, go to Workflows → Select workflow → Download → Export as JSON
```

Store the JSON file in your version control system.

## Performance Considerations

- **Execution Time:** Average 8-12 seconds per notification
- **API Rate Limits:** Monitor WooCommerce API rate limits (default: 10 requests/second)
- **Concurrency:** Safe for multiple simultaneous notifications
- **Storage:** Google Sheet rows grow by 1 per notification (negligible impact)

## Changelog

### v1.0.0 (Initial Release)
- Core workflow for missing address detection
- Integration with Claude for intelligent extraction
- WooCommerce order validation
- Email, SMS, and Slack notifications
- Google Sheets tracking
- Comprehensive error handling

## License

This workflow is provided as-is for use with your e-commerce operations. Modify as needed for your specific use case.

## Future Enhancements

Potential improvements for future versions:

1. **AI-Powered Address Correction:** Use Claude to intelligently correct partial addresses
2. **Warehouse Integration:** Direct integration with 3PL APIs instead of webhook
3. **Predictive Analytics:** ML model to predict which customers will complete address correction
4. **A/B Testing:** Test different email templates to optimize response rates
5. **Multi-Channel Escalation:** Automatic escalation to phone call if email/SMS not responded to within 24 hours
6. **International Compliance:** Address format validation for different countries
7. **Real-time Dashboard:** Live monitoring dashboard for missing address cases

---

**Questions?** Review the inline comments in the workflow JSON or check the n8n documentation for specific node configurations.
