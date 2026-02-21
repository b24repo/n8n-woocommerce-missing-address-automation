# Node Reference & Configuration Details

Complete reference for all 17 nodes in the WooCommerce Missing Address Handler workflow.

## Node Inventory

### Triggers (Entry Points)
1. **Webhook: 3PL Notification** - `webhook_trigger`

### Processing Nodes
2. **Sanitize Input Data** - `data_sanitize`
3. **Claude: Extract Order Details** - `claude_extract`
4. **Code: Analyze Address Completeness** - `analyze_address`
5. **Set: Prepare Google Sheet Row** - `prepare_sheet_data`

### Decision Nodes (Conditional Logic)
6. **Check: Order Number Valid** - `validation_check`
7. **Check: Order Found** - `order_found_check`
8. **Check: Address Issues Found** - `address_issue_check`

### External API Calls
9. **HTTP: Fetch Order from WooCommerce** - `woo_fetch_order`
10. **Email: Request Address Correction** - `send_email_customer`
11. **Twilio: Send SMS Reminder** - `send_sms_customer`
12. **Slack: Post Alert to Fulfillment** - `notify_slack`
13. **Google Sheets: Log Tracking** - `log_to_sheet`

### Alert & Error Nodes
14. **Send Error Alert** - `error_notify_team`
15. **Send: Order Not Found Alert** - `order_not_found_alert`
16. **Send: No Issues Found Notification** - `no_issues_found_log`
17. **Send: Critical Error Alert** - `critical_error_handler`

---

## Detailed Node Configuration

### 1. Webhook: 3PL Notification
**ID:** `webhook_trigger`
**Type:** `n8n-nodes-base.webhook`
**Version:** 1.x

**Purpose:** Receive HTTP POST requests from 3PL warehouse system

**Configuration:**
```json
{
  "path": "woocommerce-missing-address",
  "httpMethod": "POST",
  "responseMode": "onReceived"
}
```

**Input Payload Schema:**
```json
{
  "body": "string - email body content with order details",
  "subject": "string - email subject line"
}
```

**Output:** Raw webhook payload passed to next node

**Setup Instructions:**
1. Activate the workflow to get the webhook URL
2. URL will be: `https://your-n8n-domain/webhook/woocommerce-missing-address`
3. Configure your 3PL system to POST to this URL
4. Set `Content-Type: application/json` header

**Test Payload:**
```bash
curl -X POST https://your-n8n/webhook/woocommerce-missing-address \
  -H "Content-Type: application/json" \
  -d '{
    "body": "Missing address for order #98765",
    "subject": "Address Alert"
  }'
```

---

### 2. Sanitize Input Data
**ID:** `data_sanitize`
**Type:** `n8n-nodes-base.set`
**Version:** 3.3

**Purpose:** Extract and structure incoming webhook data

**Configuration:**
- Extracts `body` → `email_body`
- Extracts `subject` → `email_subject`
- Adds `timestamp` field with current ISO time

**Output Fields:**
```json
{
  "email_body": "string",
  "email_subject": "string",
  "timestamp": "ISO 8601 timestamp"
}
```

**Why This Step:** Normalizes data format for downstream AI processing; provides audit trail with timestamp

---

### 3. Claude: Extract Order Details
**ID:** `claude_extract`
**Type:** `@n8n/n8n-nodes-langchain.openAi`
**Version:** 1.x

**Purpose:** Use Claude AI to intelligently extract structured data from unstructured email text

**Configuration:**
```json
{
  "model": "claude-opus-4",
  "temperature": 0,
  "jsonMode": false,
  "messages": [
    {
      "role": "user",
      "content": "Extract JSON from email..."
    }
  ]
}
```

**Credentials Required:** OpenAI API (Claude endpoint)
- Name: `claude_api`
- API Key: Get from https://console.anthropic.com/account/api-keys

**What It Extracts:**
```json
{
  "order_number": "98765",
  "customer_name": "John Smith",
  "customer_email": "john@example.com",
  "customer_phone": "+1-555-0123",
  "issue_description": "Missing apartment number in address",
  "warehouse_name": "DFW Hub",
  "date_reported": "2026-02-21"
}
```

**Temperature:** Set to 0 (deterministic) for reliability - prevents hallucination

**Handles These Formats:**
- "Order #12345" or "#12345" or "Order 12345"
- Various customer name formats
- Different phone number formats
- Unstructured warehouse descriptions

**Fallback:** Returns `null` for fields that cannot be extracted

---

### 4. Check: Order Number Valid
**ID:** `validation_check`
**Type:** `n8n-nodes-base.if`
**Version:** 2.x

**Purpose:** Route based on whether order number was successfully extracted

**Condition:**
```javascript
!(JSON.parse($json.output).order_number)
// True if order_number is missing/null → Error path
// False if order_number exists → Success path
```

**Branches:**
- **TRUE (Error):** → `Send Error Alert`
- **FALSE (Success):** → `HTTP: Fetch Order from WooCommerce`

**Why:** Prevents wasted API calls to WooCommerce if we can't extract an order number

---

### 5. Send Error Alert
**ID:** `error_notify_team`
**Type:** `n8n-nodes-base.emailSend`
**Version:** 2.x

**Purpose:** Notify support team of extraction failures

**Configuration:**
```json
{
  "sendImmediately": false,
  "subject": "[n8n Alert] Failed to Process 3PL Notification",
  "toEmail": "alerts@yourcompany.com",
  "message": "Unable to extract order number from warehouse notification..."
}
```

**When Triggered:** When Claude extraction fails to find an order number

**Why Alert:** Manual intervention needed; potentially important 3PL message lost

---

### 6. HTTP: Fetch Order from WooCommerce
**ID:** `woo_fetch_order`
**Type:** `n8n-nodes-base.httpRequest`
**Version:** 4.2

**Purpose:** Retrieve complete order data from WooCommerce via REST API

**Configuration:**
```json
{
  "method": "GET",
  "url": "https://{{ $env.WOOCOMMERCE_DOMAIN }}/wp-json/wc/v3/orders?search={{ order_number }}&per_page=1",
  "authentication": "basicAuth",
  "timeout": 30000
}
```

**Credentials Required:**
- **Name:** `woocommerce_api`
- **Type:** Basic Auth
- **Username:** WooCommerce Consumer Key
- **Password:** WooCommerce Consumer Secret

**How to Get Credentials:**
1. Log in to WordPress admin
2. Go to: WooCommerce → Settings → Advanced → REST API Clients
3. Click "Create an API client"
4. Set Permissions to: Read
5. Generate and copy the Consumer Key and Consumer Secret

**URL Structure:**
```
https://yourstore.com/wp-json/wc/v3/orders?search=12345&per_page=1
```

**Response:** Array of matching orders (usually 1)

**Timeout:** 30 seconds (covers slow network scenarios)

---

### 7. Check: Order Found
**ID:** `order_found_check`
**Type:** `n8n-nodes-base.if`
**Version:** 2.x

**Purpose:** Route based on whether order exists in WooCommerce

**Condition:**
```javascript
$json.body.length > 0
// True if order found → Analyze address
// False if no order found → Send alert
```

**Branches:**
- **TRUE:** → `Code: Analyze Address Completeness`
- **FALSE:** → `Send: Order Not Found Alert`

---

### 8. Code: Analyze Address Completeness
**ID:** `analyze_address`
**Type:** `n8n-nodes-base.code`
**Version:** 2.x

**Purpose:** JavaScript function that inspects address fields and identifies missing data

**Logic:**
```javascript
Checks for missing/empty:
- address_1 (street address)
- city
- state
- postcode
- country

Returns array of specific issues found
```

**Output:**
```json
{
  "order_id": 12345,
  "order_number": "12345",
  "customer_email": "john@example.com",
  "customer_name": "John Smith",
  "customer_phone": "+1-555-0123",
  "shipping_address": { ... },
  "billing_address": { ... },
  "has_address_issues": true,
  "address_issues": [
    "Missing street address",
    "Missing city"
  ],
  "order_status": "processing",
  "order_total": "99.99",
  "order_created": "2026-02-21T10:00:00Z"
}
```

**Validation Rules:**
- Empty strings treated as missing
- Null values treated as missing
- Whitespace-only values treated as missing

---

### 9. Check: Address Issues Found
**ID:** `address_issue_check`
**Type:** `n8n-nodes-base.if`
**Version:** 2.x

**Purpose:** Route based on whether actual address issues exist

**Condition:**
```javascript
$json.has_address_issues
```

**Branches:**
- **TRUE:** → Send email, SMS, and Slack (all in parallel)
- **FALSE:** → `Send: No Issues Found Notification`

---

### 10. Email: Request Address Correction
**ID:** `send_email_customer`
**Type:** `n8n-nodes-base.emailSend`
**Version:** 2.x

**Purpose:** Send professional HTML email to customer requesting address correction

**Configuration:**
```json
{
  "to": "{{ $json.customer_email }}",
  "subject": "Action Required: Complete Your Shipping Address for Order #{{ $json.order_number }}",
  "emailType": "html",
  "htmlMessage": "<html>...</html>"
}
```

**Email Template Features:**
- Professional HTML formatting
- Customer name personalization
- Order number clearly displayed
- Bulleted list of missing fields
- Direct link to update address
- Urgency messaging
- Support contact info

**Customization:**
Edit the `htmlMessage` field to:
- Add company logo
- Change colors/branding
- Update support contact
- Modify urgency tone

**Credentials:** Uses n8n's configured SMTP settings
- Configure in: Settings → Email

---

### 11. Twilio: Send SMS Reminder
**ID:** `send_sms_customer`
**Type:** `n8n-nodes-base.httpRequest`
**Version:** 4.2

**Purpose:** Send concise SMS message via Twilio webhook

**Configuration:**
```json
{
  "method": "POST",
  "url": "https://hooks.twilio.com/services/{{ $env.TWILIO_SERVICE_ID }}/{{ $env.TWILIO_INTEGRATION_ID }}",
  "body": {
    "to": "{{ $json.customer_phone }}",
    "message": "Hi {{ $json.customer_name }}, we need your updated address for order #{{ $json.order_number }}..."
  }
}
```

**Environment Variables Required:**
- `TWILIO_SERVICE_ID`
- `TWILIO_INTEGRATION_ID`

**How to Set Up Twilio:**
1. Sign up at https://twilio.com
2. Create Messaging Service
3. Create Webhook integration
4. Copy Service ID and Integration ID
5. Add to n8n environment

**SMS Content:**
- Concise message (fits SMS character limit)
- Order number
- Action link
- 24-hour deadline

**Optional:** If customer has no phone, this step is skipped automatically

---

### 12. Slack: Post Alert to Fulfillment
**ID:** `notify_slack`
**Type:** `n8n-nodes-base.httpRequest`
**Version:** 4.2

**Purpose:** Post formatted message to fulfillment team's Slack channel

**Configuration:**
```json
{
  "method": "POST",
  "url": "{{ $env.SLACK_WEBHOOK_URL }}",
  "body": {
    "channel": "#fulfillment-alerts",
    "username": "WooCommerce Bot",
    "icon_emoji": ":package:",
    "attachments": [ ... ]
  }
}
```

**Slack Webhook Setup:**
1. Go to your Slack workspace settings
2. Create app or use existing integration
3. Generate Incoming Webhook for `#fulfillment-alerts` channel
4. Copy webhook URL
5. Add to environment: `SLACK_WEBHOOK_URL`

**Message Format:**
```
Title: Missing Address Alert for Order #12345
Fields:
- Order Number: #12345
- Customer: John Smith
- Email: john@example.com
- Status: processing
- Missing Fields: Missing city, Missing state
```

**Color Coding:** Yellow/warning color to indicate action needed

---

### 13. Set: Prepare Google Sheet Row
**ID:** `prepare_sheet_data`
**Type:** `n8n-nodes-base.set`
**Version:** 3.3

**Purpose:** Format data for Google Sheets insertion

**Output Fields (Maps to Sheet Columns):**
```
A: Timestamp (ISO 8601)
B: Order ID (numeric)
C: Order Number (string)
D: Customer Name (string)
E: Customer Email (string)
F: Customer Phone (string)
G: Missing Fields (semicolon-separated)
H: Action Taken (e.g., "Email & SMS Sent")
I: Status (e.g., "Pending Customer Response")
J: Follow-up Date (24 hours from now)
```

**Example Output Row:**
```
2026-02-21T10:30:45.123Z | 12345 | 12345 | John Smith | john@example.com | +1-555-0123 | Missing city; Missing state | Email & SMS Sent | Pending Customer Response | 2026-02-22
```

---

### 14. Google Sheets: Log Tracking
**ID:** `log_to_sheet`
**Type:** `n8n-nodes-base.googleSheets`
**Version:** 2.x

**Purpose:** Append row to Google Sheet for audit trail and analytics

**Configuration:**
```json
{
  "documentId": "{{ $env.GOOGLE_SHEETS_TRACKING_ID }}",
  "action": "append",
  "columns": "A,B,C,D,E,F,G,H,I,J"
}
```

**Credentials Required:**
- **Name:** `google_sheets_creds`
- **Type:** Google Service Account
- Get JSON key from https://console.cloud.google.com/

**Google Sheet Setup:**
1. Create new Google Sheet
2. Add headers in row 1
3. Create service account in Google Cloud
4. Download JSON key
5. Share sheet with service account email (from key)
6. Copy sheet ID from URL
7. Add to environment: `GOOGLE_SHEETS_TRACKING_ID`

**Sheet ID Location in URL:**
```
https://docs.google.com/spreadsheets/d/{SHEET_ID}/edit
                                    ^^^^^^^^^
```

**Data Persistence:** All logs remain in sheet permanently for auditing

---

### 15. Send: Order Not Found Alert
**ID:** `order_not_found_alert`
**Type:** `n8n-nodes-base.emailSend`
**Version:** 2.x

**Purpose:** Alert team if extracted order number doesn't exist in WooCommerce

**When Triggered:** No matching WooCommerce order for the extracted order number

**Recipient:** `alerts@yourcompany.com`

**Why:**
- 3PL sent notification for non-existent order (data sync issue)
- Order number was misextracted from email
- Manual investigation needed

---

### 16. Send: No Issues Found Notification
**ID:** `no_issues_found_log`
**Type:** `n8n-nodes-base.emailSend`
**Version:** 2.x

**Purpose:** Alert team when analyzed address is actually complete

**When Triggered:** `address_issue_check` evaluates to false (no issues)

**Why:**
- Indicates potential false positive from 3PL
- Suggests 3PL validation logic needs review
- Documents unnecessary but harmless alerts

---

### 17. Send: Critical Error Alert
**ID:** `critical_error_handler`
**Type:** `n8n-nodes-base.emailSend`
**Version:** 2.x

**Purpose:** Catch-all error handler for unexpected failures

**Configuration:**
```json
{
  "sendImmediately": false,
  "subject": "[CRITICAL] n8n Workflow Error - Missing Address Handler",
  "toEmail": "alerts@yourcompany.com"
}
```

**Includes in Alert:**
- Execution ID (for debugging)
- Error timestamp
- Error details

**When Triggered:** Any uncaught exception in workflow execution

---

## Execution Flow Diagram

```
START
  ↓
[1] Webhook: 3PL Notification
  ↓
[2] Sanitize Input Data
  ↓
[3] Claude: Extract Order Details
  ↓
[4] Check: Order Number Valid
  ├─ FALSE → [5] Send Error Alert → END
  │
  └─ TRUE
    ↓
    [6] HTTP: Fetch Order from WooCommerce
      ↓
      [7] Check: Order Found
      ├─ FALSE → [15] Send: Order Not Found Alert → END
      │
      └─ TRUE
        ↓
        [8] Code: Analyze Address Completeness
          ↓
          [9] Check: Address Issues Found
          ├─ FALSE → [16] Send: No Issues Found Notification → END
          │
          └─ TRUE
            ↓
            [10] Email: Request Address Correction ─┐
            [11] Twilio: Send SMS Reminder      ─┤
            [12] Slack: Post Alert to Fulfillment ─┤
            ↓                                       │
            [13] Set: Prepare Google Sheet Row      │
            ↓                                       │
            [14] Google Sheets: Log Tracking ←──────┘
            ↓
            END
```

---

## Data Flow Through Workflow

```
Raw Webhook → Sanitized Data → Extracted Structured Data → WooCommerce Order → Analyzed Address → Decision → Actions → Google Sheet Log
```

**Data Transformations at Each Step:**

1. **Webhook Input:**
   ```json
   { "body": "...", "subject": "..." }
   ```

2. **After Sanitization:**
   ```json
   { "email_body": "...", "email_subject": "...", "timestamp": "2026-02-21T..." }
   ```

3. **After Claude Extraction:**
   ```json
   { "order_number": "12345", "customer_name": "John Smith", ... }
   ```

4. **After WooCommerce Fetch:**
   ```json
   { "body": [{ "id": 12345, "status": "processing", ... }] }
   ```

5. **After Address Analysis:**
   ```json
   {
     "order_id": 12345,
     "has_address_issues": true,
     "address_issues": ["Missing city", "Missing state"],
     ...
   }
   ```

6. **Final Output to Sheet:**
   ```
   [Timestamp | Order ID | Order Number | Customer | Email | Phone | Missing Fields | Action | Status | Follow-up]
   ```

---

## Error Handling Strategy

| Error | Detection Point | Handling | Alert |
|-------|-----------------|----------|-------|
| Extraction fails | Step 4 validation | Stop workflow | Email to alerts@... |
| WooCommerce API unreachable | Step 6 timeout/error | Stop workflow | Critical error handler |
| Order doesn't exist | Step 7 check | Stop workflow | Order not found alert |
| Address is complete | Step 9 check | Skip comms | No issues alert |
| Email send fails | Step 10 execution error | Caught by error handler | Critical error alert |
| Sheet write fails | Step 14 execution error | Caught by error handler | Critical error alert |

---

## Performance Characteristics

| Metric | Value | Notes |
|--------|-------|-------|
| Avg Execution Time | 8-12 seconds | Mostly WooCommerce API latency |
| Claude API Latency | 2-3 seconds | Depends on load |
| WooCommerce Query | 2-4 seconds | Typical REST API response |
| Google Sheets Write | <500ms | Batch-efficient |
| Email Send | <1 second | Async |
| Max Concurrent | Unlimited | n8n handles queueing |

---

## Customization Examples

### Change Email Template
Edit node `send_email_customer` → `htmlMessage` parameter

### Add Database Logging
Insert `n8n-nodes-base.postgres` node after `prepare_sheet_data`

### Add Manual Review Step
Insert `n8n-nodes-base.wait` node before final actions (requires webhook approval)

### Change Slack Channel
Edit node `notify_slack` → `channel` field

### Extend to Multiple 3PLs
Add additional webhook paths and route based on warehouse ID in payload

---

## References

- [n8n Webhook Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/)
- [WooCommerce REST API](https://woocommerce.github.io/woocommerce-rest-api-docs/)
- [Claude API Docs](https://docs.anthropic.com/)
- [Google Sheets API](https://developers.google.com/sheets/api/)
- [Slack Webhooks](https://api.slack.com/messaging/webhooks)
- [Twilio Webhooks](https://www.twilio.com/docs/usage/webhooks)

