# 🚨 GitHub Issue to Slack Alert — Instant Team Notifications for New Issues

An n8n automation that watches a GitHub repository and instantly posts a formatted alert to Slack the moment a new issue is opened — so a team never has to manually check GitHub to know something needs attention.

Built entirely with **built-in n8n nodes** — GitHub Trigger, IF, Set, Slack.

---

## 🎥 Demo

[![Watch Demo Video](https://img.shields.io/badge/▶%20Watch%20Demo-Google%20Drive-2ec4a0?style=for-the-badge&logo=googledrive&logoColor=white)](https://drive.google.com/file/d/1AzIIhFfvnpQcqhooF4WSQNne93dyl00t/view?usp=sharing)

---

## 🎯 The Problem This Solves

Teams that live in Slack shouldn't have to remember to check GitHub separately to catch new issues — by the time someone notices, hours may have passed. This workflow closes that gap automatically:

- The moment a **new issue is opened** on a watched repository, a **formatted alert is posted to a Slack channel** — title, issue number, author, repo, and a direct link, all in one message.
- Other issue activity (edits, comments, closing, labeling, reopening) is **deliberately filtered out**, so the channel only lights up for genuinely new issues, not every small update.
- No manual polling, no browser tab to keep checking — it's event-driven and near-instant.

---

## 🏗️ Architecture & Flow

```
GitHub Trigger  (event: issues, webhook-based)
     │
     ▼
IF - New Issue Only
(checks $json.body.action === "opened")
     │
     ├── TRUE ───▶ Set - Format Slack Message
     │              (builds a single formatted string:
     │               repo, title, issue #, author, link)
     │                     │
     │                     ▼
     │              Slack - Send Slack Alert
     │              (posts to #all-n8n-github-alerts)
     │
     └── FALSE ──▶ (nothing connected — flow ends silently
                     for edits, closes, comments, etc.)
```

---

## ✨ Key Design Decisions

- **GitHub's webhook payload arrives nested under `$json.body`, not flat** — this instance's GitHub Trigger wraps the actual GitHub event data inside a `body` object, alongside `headers` and `query` (the same pattern hit with Gmail Trigger's `headers.subject` back in P1). Every expression in this workflow references `$json.body.action`, `$json.body.issue.title`, etc. — not the flat `$json.issue.title` that GitHub's own API docs might suggest at first glance. This was only caught by inspecting the real trigger output live, not assumed from documentation.
- **IF node filters strictly on `action === "opened"`** — the GitHub `issues` webhook event fires for many actions (opened, edited, closed, reopened, labeled, assigned, etc.), not just new issues. Without this filter, the Slack channel would get pinged for every minor update to every issue. Scoping to `"opened"` only keeps the channel meaningful.
- **Slack's "Include Link to Workflow" option turned off** — by default, the Slack node appends an "Automated with this n8n workflow" footer link (which also exposes the n8n instance's internal URL) to every message. This was disabled in `otherOptions.includeLinkToWorkflow` so the alert reads as a clean, professional message with no internal tooling exposed.
- **Message built in a separate Set node, not inline in the Slack node** — keeps the message template easy to read and edit in one place, and makes it simple to preview exactly what text will be sent before it's posted, without having to open the Slack node itself.
- **No error-handling branch** — unlike P2, this workflow has no sensitive payload at stake if a step fails (the issue itself remains safely on GitHub regardless of whether the Slack post succeeds), and the assignment spec didn't call for one. Kept intentionally simple.

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Automation engine | n8n v2.28.7 |
| Issue detection | GitHub Trigger node (webhook, OAuth2/PAT) |
| Filtering | IF node |
| Message formatting | Set node |
| Alerting | Slack node (Bot Token, free tier) |

---

## ⚙️ Setup & Installation

This setup has more moving parts than earlier projects since it involves creating both a GitHub token and a Slack app from scratch. Every step below is written out in full — nothing skipped.

### Prerequisites
- A running n8n instance, version 2.28.7 or compatible
- A GitHub account
- A Slack workspace (a free one works fine)

---

### Step 1 — Create a GitHub test repository
1. Go to [github.com/new](https://github.com/new).
2. **Repository name**: e.g. `n8n-test-repo`.
3. Check **"Add a README file"** so the repo isn't empty.
4. Click **Create repository**.
5. Note your GitHub username and exact repo name from the resulting URL — both are needed in n8n later.

---

### Step 2 — Create a GitHub Personal Access Token
1. Go to [github.com/settings/tokens](https://github.com/settings/tokens) → **Tokens (classic)** → **Generate new token (classic)**.
2. Name it, e.g. `n8n-p4-token`.
3. Set an expiration (90 days is fine for a test project).
4. Under **scopes**, check both:
   - `repo`
   - `admin:repo_hook` (required — lets n8n register the webhook automatically)
5. Click **Generate token** → copy it immediately (GitHub only shows it once).

---

### Step 3 — Create a Slack app from scratch
1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App**.
2. Choose **From scratch**.
3. **App Name**: e.g. `n8n GitHub Alerts`.
4. **Pick a workspace** to install it into.
5. Click **Create App** — this lands on the app's **Basic Information** page.

---

### Step 4 — Add the bot permission scope
1. On the app's page, find **OAuth & Permissions** in the left sidebar (under Settings/Features — scroll down if it's not immediately visible).
2. Scroll to the **Scopes** section → **Bot Token Scopes** → click **Add an OAuth Scope**.
3. Add: `chat:write`.

---

### Step 5 — Install the app and copy the bot token
1. Scroll back to the top of the same **OAuth & Permissions** page.
2. Click **Install to Workspace** (or **Reinstall to Workspace** if scopes were added after a first install).
3. Click **Allow** on the permission screen.
4. Copy the **Bot User OAuth Token** shown at the top of the page — it starts with `xoxb-...`.

---

### Step 6 — Create the Slack channel and invite the bot
1. In Slack, create a channel, e.g. `#all-n8n-github-alerts`.
2. Inside that channel, type `/invite @n8n GitHub Alerts` (use your bot's actual name) and send.
3. Confirm a system message appears: *"n8n GitHub Alerts was added to #all-n8n-github-alerts"*.

Without this step, the Slack node fails with a `not_in_channel` error even with a valid token.

---

### Step 7 — Import the workflow
1. Open n8n → **+ Create Workflow** → **Import from File**.
2. Select `P4_GitHub_Issue_Slack_Alert.json`.
3. All 4 nodes should appear pre-arranged on the canvas.

---

### Step 8 — Add credentials and values inside n8n

| Node | What to change |
|---|---|
| **GitHub Trigger** | Attach GitHub credential (Personal Access Token from Step 2); set **Repository Owner** to your GitHub username; set **Repository Name** to your real repo name |
| **Send Slack Alert** | Attach Slack credential (Bot Token from Step 5); confirm `channelId` matches your real channel name |

---

### Step 9 — Activate the workflow
Toggle the workflow **Active**. This is what actually registers the webhook on your GitHub repo — the GitHub Trigger node needs to be active (or in "Listening for test event" mode) to receive real events.

---

### Step 10 — Test it (see Verified Test Results below for the exact real result)

---

## ✅ Verified Test Results

### Test — New issue opened (only scenario tested)

A real issue was opened on the test repository via the GitHub UI:

- **Repo:** `sakthiprasanth16/n8n-test-repo-`
- **Title:** `Test issue for n8n`
- **Issue #:** `1`
- **Opened by:** `sakthiprasanth16`

The GitHub Trigger fired, correctly parsed the nested `$json.body.action`, passed the IF filter (`action === "opened"`), and the formatted alert was posted live to `#all-n8n-github-alerts`:

```
🚨 New GitHub Issue Opened

Repo: sakthiprasanth16/n8n-test-repo-
Title: Test issue for n8n
Issue #: 1
Opened by: sakthiprasanth16
Link: https://github.com/sakthiprasanth16/n8n-test-repo-/issues/1
```

Slack also auto-unfurled the GitHub link into a native preview card underneath the message, showing the issue title, number, and repo — confirmed visually in the actual channel.

This confirms the full chain — GitHub Trigger → IF → Set → Slack — works correctly end to end, with a real issue triggering a real, correctly formatted Slack message and no leftover "Automated with this n8n workflow" footer.

*(Only the "opened" / true-branch path was tested live; the false branch — e.g. editing or closing an existing issue producing no Slack message — was not separately verified in this session, but follows directly from the same IF condition already confirmed to route correctly on "opened".)*

---

## 🛠️ Common Issues

| Problem | Fix |
|---|---|
| `Problem running workflow — check that the repository exists and that you have permission to create the webhooks this node requires` | Caused by the GitHub Trigger node still holding placeholder values for Repository Owner/Name after import, and/or the repo not actually existing yet on GitHub. Fixed by creating the real repo first, then explicitly re-selecting the correct owner and repo in the node's fields (not just typing over the placeholder text). |
| IF node always evaluates false, even on a genuine "opened" issue | The workflow initially referenced `$json.action` and `$json.issue.title` directly. The real trigger output nests everything under `$json.body` (alongside `headers` and `query`). Fixed by updating every expression to `$json.body.action`, `$json.body.issue.title`, etc. |
| Slack message includes an unwanted "Automated with this n8n workflow" footer link | This is a default Slack node option, not a bug. Fixed by adding `includeLinkToWorkflow: false` under the Slack node's Other Options. |
| Slack node fails with a channel-not-found or permission error | The bot must be explicitly invited to the target channel with `/invite @your-bot-name` — being an app installed to the workspace is not the same as being a member of a specific channel. |

---

## 📁 Project Structure

```
P-4-GitHub-Issue-Slack-Alert/
├── P4_GitHub_Issue_Slack_Alert.json
├── README.md
└── demo.mp4
```

---

## 👨‍💻 Author
**Sakthi Prasanth**

Project: GitHub Issue to Slack Alert — Instant Team Notifications for New Issues

Built with n8n, GitHub, and Slack
