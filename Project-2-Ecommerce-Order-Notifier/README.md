# 🛒 E-Commerce Order Notifier — Webhook to Telegram + Gmail Automation

An n8n automation that receives new orders via a webhook, instantly alerts the shop owner on Telegram, sends the customer a confirmation email, and — if anything fails — automatically logs the exact failure with the real order data to a Google Sheet, so nothing is ever lost silently.

Built entirely with **built-in n8n nodes** — Webhook, Set, Telegram, Gmail, Respond to Webhook, Error Trigger, Google Sheets.

---

## 🎥 Demo

[![Watch Demo Video](https://img.shields.io/badge/▶%20Watch%20Demo-Google%20Drive-2ec4a0?style=for-the-badge&logo=googledrive&logoColor=white)](https://drive.google.com/file/d/1Wl3MaK8_HIljf1q9j9HIK6vx95FF9ZRk/view?usp=sharing)

---

## 🎯 The Problem This Solves

An online shop owner needs to know the moment a new order comes in, and the customer needs an immediate confirmation — but manual checking or delayed batch emails aren't good enough for a live storefront. This workflow automates both notifications from a single incoming order event:

- The moment an order webhook fires, the **shop owner is alerted on Telegram** with the order ID, product, total, and customer email — visible instantly, even from a phone.
- The **customer receives an automatic confirmation email** with their order details, without any manual step.
- If the confirmation email **can't be sent** (e.g. the order data is missing a customer email), the workflow **doesn't fail silently** — it captures exactly which node failed, why, and what the actual order payload looked like, and logs it to a Google Sheet for review.
- The original caller (e.g. a real checkout system) always gets a webhook response, so the integration behaves predictably either way.

---

## 🏗️ Architecture & Flow

This is a **single n8n workflow file** with one main flow and two failure-safety layers.

```
Webhook - Receive Order  (POST /order-notifier)
     │
     ▼
Set - Extract Order Fields
(order_id, customer_email, product_name, total
 pulled from the JSON body)
     │
     ▼
Telegram - Notify Shop Owner
(sends formatted alert regardless of whether
 later steps succeed)
     │
     ▼
Gmail - Send Customer Confirmation
     │
     ├── SUCCESS (output 0) ──▶ Respond to Webhook
     │                          (returns {"status":"success", ...})
     │
     └── FAILURE (output 1) ──▶ Set - Format Gmail Error
                                 (pulls the REAL order data straight
                                  from the Set node by name, so the
                                  failure log always has real values —
                                  not just error metadata)
                                        │
                                        ▼
                              Google Sheets - Log Failure


Error Trigger - Catch Failures  (separate, workflow-level safety net —
                                  fires for any OTHER unexpected error,
                                  e.g. Telegram itself failing)
     │
     ▼
Set - Format Error Details
(generic error info: timestamp, failed node,
 error message — no item-level payload, since
 Error Trigger genuinely doesn't have access to it)
     │
     ▼
Google Sheets - Log Failure   (same sheet, same node, as above)
```

---

## ✨ Key Design Decisions

- **Two separate error-capture paths, not just one** — n8n's global Error Trigger only receives execution *metadata* (which node failed, what the error message was), not the actual data that was flowing through the workflow at the time. So relying on it alone means every failure log entry would show an empty payload. To fix this, the Gmail node's own **error output** (`onError: continueErrorOutput`) is used for its specific failure case, feeding a dedicated Set node that explicitly re-fetches the real order data from `Set - Extract Order Fields` by name — guaranteeing the logged payload is never empty, even though the error branch's own data doesn't carry it forward automatically.
- **Node-name references (`$('Node Name').item.json...`) instead of `$json` after Telegram** — Telegram's own response (message ID, chat info, etc.) completely replaces `$json` for anything downstream of it. Referencing `Set - Extract Order Fields` explicitly by name in both the Gmail node and the Respond to Webhook node was the fix for what looked like a "missing customer_email" bug, but was actually just reading from the wrong node's output.
- **Error Trigger only fires on Production executions, not Test/manual runs** — this is a real, documented n8n behavior discovered while testing: running the workflow via "Listen for Test Event" never triggers the linked Error Workflow, even when a node genuinely fails. Testing the failure path properly requires publishing the workflow and sending requests to the **Production URL**, not the Test URL.
- **Workflow set as its own Error Workflow** — under workflow Settings → Error Workflow, this workflow points to itself, which is what allows the standalone Error Trigger node to fire at all. This is easy to miss, since importing a workflow JSON does not carry this setting over automatically.
- **Telegram notification happens before Gmail, and doesn't depend on it** — since Telegram is earlier in the chain, the shop owner still gets notified of every order attempt, even one with incomplete data, before Gmail's own validation potentially fails downstream. This means the owner sees "something is wrong with this order" even when the customer-facing email couldn't go out.
- **Respond to Webhook only fires on the success path** — since it's connected after Gmail's success output, a genuine caller (e.g. a checkout system) only gets an acknowledgment once the order has been fully processed, not a false-positive "success" if a downstream step actually failed.

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Automation engine | n8n v2.28.7 |
| Order intake | Webhook node (native, POST) |
| Owner alerting | Telegram node (Bot API, free tier) |
| Customer email | Gmail node (OAuth2, free tier) |
| Failure logging | Google Sheets node (OAuth2, free tier) |
| Error handling | Error Trigger + node-level error output (`onError: continueErrorOutput`) |

---

## ⚙️ Setup & Installation

This is a single **n8n workflow file** — no repo to `git clone`. Set up the accounts below, import the workflow, and wire up your own credentials.

### Prerequisites
- A running n8n instance (self-hosted or cloud), version 2.28.7 or compatible
- A Telegram account
- A Gmail account (OAuth2)
- A Google account with access to Google Sheets

---

### Step 1 — Create your Telegram Bot
1. Open Telegram → search **`@BotFather`** → **Start**.
2. Send `/newbot`.
3. Give it a display name (e.g. "My Shop Order Bot").
4. Give it a username ending in `bot` (e.g. `myshoporder_bot`).
5. Copy the **API Token** BotFather returns — looks like `123456789:AAExampleTokenStringHere`.

### Step 2 — Get your Telegram Chat ID
1. Search your new bot's username in Telegram → **Start** (send it any message — required before it can message you first).
2. Visit this URL in your browser, replacing `<YOUR_TOKEN>`:
   ```
   https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates
   ```
3. Find `"chat": { "id": ... }` in the JSON response and copy that number — this is your `chatId`.

---

### Step 3 — Create the Failure Log Google Sheet
1. Go to [sheets.google.com](https://sheets.google.com) → **Blank spreadsheet**.
2. Rename it, e.g. `Order Notifier - Failure Log`.
3. Keep the tab named `Sheet1`.
4. In **row 1**, add these exact column headers:
   ```
   Timestamp | Failed Node | Error Message | Order Payload
   ```
5. Copy the **Sheet ID** from the URL (between `/d/` and `/edit`).

---

### Step 4 — Import the workflow
1. Open n8n → **+ Create Workflow** → **Import from File**.
2. Select `P2_Ecommerce_Order_Notifier_v2.json`.
3. All 9 nodes should appear pre-arranged on the canvas.

---

### Step 5 — Add credentials and values inside n8n

| Node | What to change |
|---|---|
| **Telegram - Notify Shop Owner** | Attach Telegram credential (Bot Token from Step 1); replace `chatId` with your real Chat ID from Step 2 |
| **Gmail - Send Customer Confirmation** | Attach Gmail OAuth2 credential |
| **Google Sheets - Log Failure** | Attach Google Sheets OAuth2 credential; paste your Sheet ID from Step 3 into `documentId`; confirm `sheetName` = Sheet1 |
| **Webhook - Receive Order** | No changes needed — you'll copy its URL after publishing |
| **Error Trigger - Catch Failures** | No node-level changes, but see Step 6 below |

---

### Step 6 — Link the workflow as its own Error Workflow
1. Click **⋯** (top right of canvas) → **Settings**.
2. Find **Error Workflow** → select this same workflow from the dropdown.
3. Click **Save**.

Without this step, the standalone Error Trigger branch will never fire.

---

### Step 7 — Publish the workflow
Toggle **Active** / click **Publish** — the webhook only accepts real traffic once published.

---

### Step 8 — Get your webhook URL
Open **Webhook - Receive Order** node → copy the **Production URL** (looks like `https://your-instance.app.n8n.cloud/webhook/order-notifier`).

---

### Step 9 — Test it (see Verified Test Results below for exact payloads and expected outcomes)

---

## ✅ Verified Test Results

The workflow was tested live end to end, including both the success path and the deliberate failure path.

### Test 1 — Valid order (happy path)

Sent via Postman to the Production URL:
```json
{
  "order_id": "ORD-4002",
  "customer_email": "prasanth16101k99@gmail.com",
  "product_name": "Cricket Bat",
  "total": "12000"
}
```

**Telegram result** — received instantly:
```
🛒 New Order Received!

Order ID: ORD-4002
Product: Cricket Bat
Total: ₹12000
Customer: prasanth16101k99@gmail.com
```

Postman returned `200 OK` with a success JSON, and the customer confirmation email was sent to the real address — confirming the full chain (Webhook → Set → Telegram → Gmail → Respond to Webhook) works correctly end to end.

### Test 2 — Malformed order (deliberate failure path)

Sent via Postman to the Production URL, deliberately missing `customer_email` and `total`:
```json
{
  "order_id": "ORD-4001",
  "product_name": "Payload Fix Test"
}
```

**Telegram result** — still received (Telegram doesn't require the missing fields):
```
🛒 New Order Received!

Order ID: ORD-4001
Product: Payload Fix Test
Total: ₹
Customer:
```

**Gmail node result** — failed as expected, with:
```
Cannot read properties of null (reading 'split') (item 0)
```
This happened because `customer_email` was genuinely `null`, and Gmail tried to validate it as an email address.

**Google Sheets - Log Failure result** — a new row was captured with the real order data intact:

| Timestamp | Failed Node | Error Message | Order Payload |
|---|---|---|---|
| 2026-07-12 2:35:17 | Gmail - Send Customer Confirmation | Cannot read properties of null (reading 'split') (item 0) | `{"order_id":"ORD-4001","customer_email":null,"product_name":"Payload Fix Test","total":null}` |

This confirms the error-output fix works exactly as intended: even though the failure happened inside Gmail, the actual order data (including which fields were null) was still captured accurately in the log — not an empty `{}`.

### Test 3 — Confirming the Error Trigger fires only on Production runs

An earlier attempt to trigger the failure branch using "Listen for Test Event" (Test URL) produced a Gmail failure visible on the canvas, but **no row appeared in the Failure Log sheet**. Once the same malformed payload was resent to the **Production URL** instead, the Executions tab showed two separate execution entries — the main flow (ending in a Gmail failure) and a second, independent execution starting from the Error Trigger node — and the failure log row appeared correctly. This confirmed the Error Workflow link only activates for live/production traffic, not manual test runs.

---

## 🛠️ Common Issues

| Problem | Fix |
|---|---|
| Gmail node fails with `Cannot read properties of undefined (reading 'split')` even on a valid-looking test payload | The Gmail node's To/Subject/Message fields were referencing `$json`, which by that point in the chain had already been overwritten by the Telegram node's own response data (message ID, chat info, etc.), not the original order fields. Fix by explicitly referencing the earlier node by name: `{{$('Set - Extract Order Fields').item.json.customer_email}}`. |
| A malformed test payload doesn't produce any row in the Failure Log sheet, even though the Gmail node visibly failed on the canvas | The workflow was tested using "Listen for Test Event" (Test URL). n8n's linked Error Workflow only fires for Production executions, not manual/test runs. Publish the workflow and resend the request to the **Production URL** instead. |
| Failure log rows show `Order Payload: {}` instead of the real order data | The original Error Trigger-only design relied on `$json.execution.data`, which does not actually contain the failed node's input data — only execution metadata. Fixed by giving the Gmail node its own error output (`onError: continueErrorOutput`) wired to a dedicated Set node that explicitly re-fetches the real values from `Set - Extract Order Fields` by name. |
| Respond to Webhook returns `undefined` for `order_id` | Same root cause as the Gmail issue above — update its response body expression to also reference `$('Set - Extract Order Fields').item.json["order_id"]` instead of plain `$json`. |
| Telegram message sends fine even when the order is missing required fields | Expected — Telegram is earlier in the chain than Gmail and doesn't require `customer_email` or `total` to function, so it always fires regardless of what happens downstream. This is useful: the shop owner still gets notified something arrived, even if it's incomplete. |
| Error Trigger node never seems to do anything, no matter what fails | It must be manually linked: workflow Settings → **Error Workflow** → select this same workflow. This setting does not carry over when importing a workflow JSON and must be re-set after every fresh import. |

---

## 📁 Project Structure

```
Project-2-Ecommerce-Order-Notifier/
├── P2_Ecommerce_Order_Notifier_v2.json
├── README.md
└── demo.mp4
```

This project is a single importable workflow file — there is only one JSON to import, containing both the main order-processing flow and both error-handling branches.

---

## 👨‍💻 Author
**Sakthi Prasanth**

Project: E-Commerce Order Notifier — Webhook to Telegram + Gmail Automation

Built with n8n, Telegram, Gmail, and Google Sheets
