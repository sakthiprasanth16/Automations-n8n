# 🤖 AI Customer Support Bot — Instant AI Replies with a Human Fallback Safety Net

An n8n automation that receives a customer's support question via webhook, gets an instant AI-generated answer from Google Gemini, and replies directly to the caller — with an automatic human-agent fallback if the AI call fails or returns no usable answer.

Built entirely with **built-in and native AI n8n nodes** — Webhook, Google Gemini (Message a Model), IF, Gmail, Respond to Webhook.

---

## 🎥 Demo

[![Watch Demo Video](https://img.shields.io/badge/▶%20Watch%20Demo-Google%20Drive-2ec4a0?style=for-the-badge&logo=googledrive&logoColor=white)](https://drive.google.com/file/d/1cg25TdujU5S_QKwIt-vH4bJxd0Y1uDPa/view?usp=sharing)

---

## 🎯 The Problem This Solves

Customers expect an immediate response, but staffing a support desk around the clock isn't realistic for every team. This workflow gives every customer a fast, useful answer automatically, without ever leaving them with silence if something goes wrong:

- A customer's question, submitted via webhook, is answered **instantly by AI**, with a reply written in a professional, helpful support-agent tone.
- If the AI call **fails outright** (API error, invalid model, network issue) or **returns no usable text**, the customer is never left hanging — they immediately get a friendly holding message instead of an error.
- At the same time, a **human agent is automatically emailed** the original question and a timestamp, so a real follow-up can happen without the customer needing to ask twice.

---

## 🏗️ Architecture & Flow

```
Webhook - Receive Question  (POST /support-bot, { "question": "..." })
     │
     ▼
Google Gemini - Message a Model
(sends a system prompt defining a support-agent persona,
 plus the customer's question, to Gemini)
     │
     ├── SUCCESS (output 0) ──▶ IF - Check Empty Reply
     │                          (checks if the returned text is blank)
     │                                │
     │                                ├── FALSE (has real text) ──▶ Respond With AI Answer
     │                                │                              (returns Gemini's reply to the caller)
     │                                │
     │                                └── TRUE (empty text) ──▶ Notify Human Agent
     │
     └── ERROR (output 1) ───────────────────────────────────▶ Notify Human Agent
                                                                       │
                                                                       ▼
                                                              Respond With Fallback Message
                                                              (tells the caller a human will follow up)
```

---

## ✨ Key Design Decisions

- **Native Gemini node used instead of a raw HTTP Request** — the assignment spec was originally written for OpenAI via HTTP Request, but this project uses Google Gemini per project-wide requirements. Rather than manually building the API call, n8n's native `Message a Model` (Google Gemini) node handles authentication and request formatting directly.
- **Both failure modes route to the same fallback path** — the assignment requires a fallback "if the AI call fails **or** returns an empty response." These are genuinely two different failure conditions, caught two different ways: the Gemini node's own **error output** (enabled via Settings → On Error → "Continue (using error output)") catches outright API failures, while a separate **IF node** checks specifically for blank/empty response text on the success path. Both converge on the same "Notify Human Agent" node, so the customer experience is identical regardless of which failure actually happened.
- **Gemini's real response shape required a live test to confirm** — the response text isn't a flat string at `$json.content`; it's nested at `$json.content.parts[0].text`, alongside metadata like `role`, `finishReason`, and `thoughtSignature`. This was only discovered by inspecting the node's real output during testing, not assumed from documentation, and both the IF node and the success-response node were corrected to use the real path.
- **The Gmail fallback re-fetches the original question by node name** — `$('Receive Question').item.json.body.question`, not `$json`, since by the time execution reaches "Notify Human Agent," `$json` reflects whatever the Gemini node (or its error) last output, not the original webhook payload — same defensive pattern used in P2 and P4.
- **The customer always receives a response, either way** — success returns `{"status":"success","reply":"<Gemini's actual answer>"}`, and any failure returns `{"status":"fallback","reply":"Thanks for reaching out! Our support team has received your question and one of our executives will get back to you shortly."}` — never a raw error, never silence.

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Automation engine | n8n v2.28.7 |
| Question intake | Webhook node (POST, JSON body) |
| AI response | Google Gemini node (`Message a Model`, native n8n AI integration) |
| Failure detection | IF node + Gemini node's built-in error output |
| Human fallback notification | Gmail node (OAuth2, free tier) |
| Response delivery | Respond to Webhook node |

---

## ⚙️ Setup & Installation

### Prerequisites
- A running n8n instance, version 2.28.7 or compatible
- A Gemini API key (Google AI Studio, free tier)
- A Gmail account (OAuth2)

---

### Step 1 — Import the workflow
1. Open n8n → **+ Create Workflow** → **Import from File**.
2. Select `P7_AI_Support_Bot.json`.
3. All 6 nodes should appear pre-arranged on the canvas.

---

### Step 2 — Configure the Gemini node
1. Open **Message a Model**.
2. Attach your Gemini credential.
3. Confirm the model selector shows a valid, currently-available model.
4. Go to the **Settings** tab of this node → confirm **"On Error"** is set to **"Continue (using error output)"** — this is required for the fallback logic to work, and does not carry over reliably on every import.

---

### Step 3 — Configure the Gmail node
1. Open **Notify Human Agent**.
2. Attach your Gmail OAuth2 credential.
3. Replace `PASTE_SUPPORT_AGENT_EMAIL_HERE` in the `sendTo` field with the real support agent's email address.

---

### Step 4 — Publish the workflow
Toggle **Active** — the webhook only accepts real traffic once published.

---

### Step 5 — Get your webhook URL
Open **Receive Question** → copy the **Production URL**.

---

### Step 6 — Test it (see Verified Test Results below)

---

## ✅ Verified Test Results

### Test 1 — Valid question, successful AI response

Sent via Postman to the Production URL:
```json
{ "question": "What are your business hours?" }
```

**Gemini's real response** (confirmed from the node's live output):
> Our customer support team is available Monday through Friday, from 9:00 AM to 5:00 PM (EST). If you reach out outside of these hours, we will respond as soon as we return the next business day.

The `Check Empty Reply` IF node correctly evaluated this as non-empty and routed to **Respond With AI Answer**, returning the reply directly to the caller. This confirmed the full success chain — Webhook → Gemini → IF → Respond to Webhook — works correctly end to end, using the corrected `$json.content.parts[0].text` field path.

### Test 2 — API failure (invalid model, forces fallback)

The Gemini node's model was temporarily set to an invalid model to force a real API failure, then a normal question was sent:
```json
{ "question": "What is your return policy?" }
```

**Result:** The Gemini node's error output correctly fired (thanks to `onError: continueErrorOutput`), routing directly to **Notify Human Agent**, bypassing the IF node entirely as designed.

**Fallback email received** — real inbox, subject **"Response Failure : Unanswered Customer Support Question"**:
> A customer question could not be answered automatically and needs human follow-up. Question: What is your return policy? Submitted at: 7/17/2026, 2:36:53 AM

This confirmed the failure path works correctly end to end with a genuine API error: the original question was preserved accurately (via the node-name reference back to "Receive Question"), a real timestamp was generated, and the human agent received exactly the information needed to follow up — with the customer still receiving a graceful fallback response rather than a raw error.

*(The model was reverted to the correct real model immediately after this test.)*

---

## 🛠️ Common Issues

| Problem | Fix |
|---|---|
| `Check Empty Reply` fails with `Wrong type: '[object Object]' is an object but was expecting a string` | Gemini's real response text is nested at `$json.content.parts[0].text`, not directly at `$json.content` as initially assumed. `content` itself is an object containing metadata (`role`, `finishReason`, `thoughtSignature`) alongside the actual `parts` array. Fixed by updating both the IF node's condition and the success response body to reference the correct nested path. |
| Gemini node fails the entire workflow on an API error instead of triggering the fallback email | The native Gemini node's error-output behavior isn't enabled by default. Fixed by opening the node's **Settings** tab → **On Error** → selecting **"Continue (using error output)"**, then manually wiring its second (error) output to the "Notify Human Agent" node. |
| Native `Message a Model` node was initially configured with a raw JSON request body as its `content` field | This node expects the plain question text as `content`, not a manually constructed API request body — that structure is built internally by the node itself. Fixed by simplifying `content` to just `{{ $json.body.question }}` and moving the system prompt into a separate message with `role: "system"`. |

---

## 📁 Project Structure

```
P-7-AI-Customer-Support-Bot/
├── P7_AI_Support_Bot.json
├── README.md
└── AI_Customer_Support_Bot.mp4
```

---

## 👨‍💻 Author
**Sakthi Prasanth**

Project: AI Customer Support Bot — Instant AI Replies with a Human Fallback Safety Net

Built with n8n, Google Gemini, and Gmail
