# 🤖 Live Data AI Assistant — Telegram Bot

An AI assistant on Telegram that **knows when it doesn't know something live** — it automatically detects whether your question needs real-time internet data (news, prices, jobs) or general knowledge, fetches live data through Apify when needed, and always replies with a clean, human-readable, bullet-formatted answer.

---

## 🎥 Demo

[![Watch Demo Video](https://img.shields.io/badge/▶%20Watch%20Demo-Google%20Drive-2ec4a0?style=for-the-badge&logo=googledrive&logoColor=white)](https://drive.google.com/file/d/1uKxg3pObXB_IsqKqZ9dH5K89Tx9Sao9s/view?usp=drive_link)

---

## 🎯 The Problem This Solves

Most simple chatbots fall into one of two traps:

1. **Pure LLM bots** — great at explaining concepts, but confidently wrong (or just outdated) the moment you ask "what's today's AI news" or "current iPhone price," since the model has no live internet access.
2. **Pure scraper bots** — can fetch live data, but dump raw, unreadable scraped text/HTML at the user with no synthesis.

This assistant solves both by combining them properly:

- A **Gemini-powered classifier** looks at every incoming message and decides: does this need *live* data, or can it be answered from general knowledge — without any hardcoded keyword matching.
- **Live** questions are routed to **Apify** to fetch actual current search data, which is converted into clean structured JSON (title, description, source, URL, date) and handed to Gemini to synthesize into a natural, bullet-pointed answer — never raw scraped data.
- **General** questions go straight to Gemini for a direct answer — no wasted API calls to Apify.
- A **typing indicator** shows in Telegram while the assistant is working, so the wait never feels broken.
- Every interaction — live or general, success or failure — is logged to Google Sheets with an IST timestamp, and every realistic failure point (Apify timeout, empty results, Gemini rate-limit at any of the three AI calls) has its own graceful fallback message, so the bot never leaves the user hanging with silence.

---

## 🏗️ Architecture & Flow

```
Telegram User
      │
      ▼
Telegram Trigger (n8n)
      │
      ▼
Extract User Query  (pulls clean user_query, chat_id, user_first_name)
      │
      ▼
Show Typing Indicator  (Telegram "typing..." while the pipeline runs)
      │
      ▼
Gemini Query Classifier   →  returns { route: LIVE | GENERAL, category }
      │   (on failure → Handle Classifier Error)
      ▼
Parse Classifier Output  (safely parses Gemini's JSON reply, with fallback)
      │
      ▼
   IF Live or General
      │
      ├── TRUE (LIVE) ──────────────────────────────────────────┐
      │                                                         │
      │   Call Apify (Google Search Scraper)                    │
      │        │  (on failure → Handle Apify Error)             │
      │        ▼                                                │
      │   Format Structured JSON  (raw results → title/desc/etc)│
      │        ▼                                                │
      │   Check Has Results                                     │
      │        │  (no results → Handle No Results)              │
      │        ▼                                                │
      │   Gemini Summary  (structured data → readable, bulleted │
      │                     answer — never raw scraped data)    │
      │        │  (on failure → Handle Gemini Error - Live)     │
      │        ▼                                                │
      │   Extract Final Response (Live)                         │
      │                                                         │
      └── FALSE (GENERAL) ──────────────────────────────────────┘
              │
              Gemini General Answer
                   │  (on failure → Handle Gemini Error - General)
                   ▼
              Extract Final Response (General)
                                                          │
                     ┌────────────────────────────────────┘
                     ▼
       ┌─────────────┴─────────────┐
       ▼                           ▼
Send Telegram Reply       Append Row to Google Sheet
(markdown symbols          (Timestamp in IST, User Query,
stripped for safe          Category, Live/General, Summary,
delivery)                  Final Response, Status)
```

**Every fallback/error node above converges back into the same two final nodes** (Telegram Reply + Sheets Logging) — so no matter which path fires (success or one of the 6 error branches), the user always gets a reply and every interaction is always logged with a clear Status (`SUCCESS`, `ERROR`, or `NO_RESULTS`).

---

## ✨ Key Design Decisions

- **AI-based classification, not keyword matching** — a single Gemini call returns both the route (LIVE/GENERAL) and a specific category (NEWS, PRICE, JOBS, STOCK, SEARCH, TECH, GITHUB, WEATHER) in one structured JSON response, keeping API usage efficient.
- **Structured data before summarization** — Apify's raw search results are always converted into a clean `{title, description, source, url, date}` shape before being handed to Gemini. Gemini is never shown raw HTML or unprocessed scrape output.
- **Shared reply/logging endpoints** — both the LIVE and GENERAL branches, and all 6 error-handling branches, converge into one shared "Send Telegram Reply" node and one shared "Append to Sheet" node, using `$json` (not hardcoded node references) so the same two nodes work correctly regardless of which path executed.
- **Graceful degradation everywhere an external call can fail** — Apify call, Apify empty-results case, and both Gemini calls (classifier, and the LIVE/GENERAL answer generation) each have dedicated error branches with user-friendly fallback messages, instead of the workflow crashing silently.
- **Telegram-safe text sanitization** — Gemini's raw output can contain markdown characters (`*`, `_`, `` ` ``) that break Telegram's message parser; these are stripped before sending, while `•` bullet characters (explicitly requested in the AI prompts) are preserved for readable, scannable formatting.

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Chat interface | Telegram Bot API |
| Automation engine | n8n (self-hosted via Docker) |
| Live data fetching | Apify — Google Search Results Scraper actor |
| AI / LLM | Google Gemini 2.5 Flash Lite (classification, summarization, general answers) |
| Logging | Google Sheets (OAuth2) |
| Local → public tunnel | ngrok (static domain) |
| Containerization | Docker |

---

## ⚙️ Setup & Installation

This is an **n8n workflow**, not a coded application — there's no repo to `git clone`. Instead, you'll set up a handful of free accounts, run n8n locally via Docker, import the workflow, and wire up your own credentials. Follow every step in order; skipping steps is the most common cause of confusing errors later.

### Prerequisites
- A computer with [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
- A Telegram account
- A Google account (for Gemini + Sheets)
- A free [Apify](https://apify.com/) account
- A free [ngrok](https://ngrok.com/) account

---

### Step 1 — Create your Telegram Bot

1. Open Telegram, search for **`@BotFather`**, tap **Start**.
2. Send `/newbot`.
3. Give it a display name (anything), then a username ending in `bot` (e.g. `my_live_assistant_bot`).
4. BotFather replies with an **API Token** — a long string like `7123456789:AAExample...`. Save this somewhere safe. You'll need it in Step 7.

---

### Step 2 — Get a free Gemini API key

1. Go to [aistudio.google.com](https://aistudio.google.com/) → sign in with Google.
2. Left sidebar → **Get API key** → **Create API key** → (if prompted) **Create API key in new project**.
3. Copy the key (starts with `AIza...`). No billing account should be required — if Google prompts for one, stop and check you're using AI Studio, not Vertex AI.
4. Confirm **Gemini 2.5 Flash Lite** is available as a model option — this is the model used throughout the workflow.

---

### Step 3 — Set up Apify

1. Go to [apify.com](https://apify.com/) → sign up (free tier, no card required).
2. Go to **Store**, search `Google Search Scraper`, open the one published by **Apify** (official, blue badge) — full name **"Google Search Results Scraper"**.
3. Click **Try for free** — this adds it to your account.
4. Note the **Actor ID** shown in the URL or the actor's **API** tab.
5. Get your **API Token**: profile icon (top right) → **Settings** → **Integrations** → copy the Personal API token.
6. Optional: run one manual test search on the actor's Input page to confirm it returns real results before wiring it into n8n.

---

### Step 4 — Create your Google Sheet (for logging)

1. Go to [sheets.google.com](https://sheets.google.com/) → create a blank sheet.
2. Rename it, e.g. `AI_Assistant_Logs`.
3. In **Row 1**, add exactly these 7 headers, one per column (A1–G1):
   ```
   Timestamp | User Query | Category | Live/General | Summary | Final Response | Status
   ```
4. Keep this tab open — you'll select it from a dropdown later, no need to copy any ID manually.

---

### Step 5 — Run n8n locally via Docker

Open a terminal (PowerShell on Windows, Terminal on Mac) and run:

```bash
docker run -d --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

- `-d` runs it in the background (persistent — survives closing the terminal, restartable anytime from Docker Desktop).
- `-v n8n_data:/home/node/.n8n` stores all your workflows/credentials in a Docker volume, so nothing is lost between restarts.

Open **http://localhost:5678** in your browser and create your local n8n owner account (email/name/password — this stays on your machine only).

> ⚠️ **This alone is not enough.** Telegram requires a public **HTTPS** URL to deliver messages — `localhost` is not reachable from the internet. Continue to Step 6.

---

### Step 6 — Expose n8n publicly with ngrok

Telegram (and Google's OAuth) both need a real public URL pointing at your local n8n instance.

1. Download and install ngrok from [ngrok.com/download](https://ngrok.com/download).
2. Sign up on the ngrok dashboard, go to **Your Authtoken**, copy it.
3. In a terminal:
   ```bash
   ngrok config add-authtoken YOUR_TOKEN_HERE
   ```
4. Reserve a **free static domain** (Dashboard → Domains → New Domain) — this gives you a permanent URL like `your-name-123.ngrok-free.dev` that never changes between sessions. Without a static domain, you get a new random URL every restart, which breaks your saved Telegram webhook and Google OAuth redirect URI each time.
5. Start the tunnel:
   ```bash
   ngrok http --domain=your-static-domain.ngrok-free.dev 5678
   ```
   Keep this terminal window open while working.
6. Re-create your n8n container so it knows its own public address, using `-e WEBHOOK_URL`:
   ```bash
   docker rm -f n8n
   docker run -d --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n -e WEBHOOK_URL=https://your-static-domain.ngrok-free.dev/ docker.n8n.io/n8nio/n8n
   ```
7. From now on, access n8n at `https://your-static-domain.ngrok-free.dev` (not `localhost`) — this is what makes Telegram and Google able to actually reach it.

**Daily startup routine after initial setup:**
- Start ngrok: `ngrok http --domain=your-static-domain.ngrok-free.dev 5678` (must be run every session — this is the one command you'll always need)
- Start n8n: open Docker Desktop → Containers → click ▶️ Play next to `n8n` (only if it shows stopped)
- Open `https://your-static-domain.ngrok-free.dev` in your browser

> Both ngrok and the n8n container must be running at the same time for the bot to respond to real Telegram messages. If either stops, the bot goes silent with no visible error until both are running again.

---

### Step 7 — Add credentials inside n8n

Go to **Credentials** (left sidebar) → **+ Add Credential** for each of the following:

**Telegram API**
- Search "Telegram", paste your Bot Token from Step 1.

**Google Gemini(PaLM) Api**
- Search "Google Gemini", paste your Gemini API key from Step 2.

**Apify**
- Search "Apify", paste your API Token from Step 3.

**Google Sheets OAuth2 API**
- Search "Google Sheets".
- This requires its own Google Cloud OAuth Client (one-time setup):
  1. Go to [console.cloud.google.com](https://console.cloud.google.com/) → create a new project.
  2. Enable **Google Sheets API** and **Google Drive API** (search bar → find each → Enable). Both are required — the Sheets node uses Drive API to list/browse files.
  3. Go to **APIs & Services → OAuth consent screen** → set User Type to **External** → fill app name + your email → **Save and Continue** through the screens → add your own email under **Test users**.
  4. Go to **APIs & Services → Credentials → + Create Credentials → OAuth client ID** → Application type **Web application**.
  5. Under **Authorized redirect URIs**, add exactly the "OAuth Redirect URL" shown in n8n's Google Sheets credential screen — it looks like:
     ```
     https://your-static-domain.ngrok-free.dev/rest/oauth2-credential/callback
     ```
  6. Copy the **Client ID** and **Client Secret** into n8n's credential screen, Save, then **Sign in with Google**.
  7. If Google shows "app not verified" — click **Advanced → Go to [app name] (unsafe)**. This is expected and safe for a personal, unpublished OAuth app.

---

### Step 8 — Import the workflow

1. In n8n, click **+ Create Workflow**.
2. Import the workflow JSON (from this repo's `Workflow/` folder) — paste it directly onto the canvas (Ctrl+V), or use the ⋯ menu → Import from File.
3. Go through each node and reselect the correct credential from the dropdowns — credential IDs don't carry over between n8n instances, so every node needs its credential re-linked once.
4. Open the **Apify** node → confirm the Actor field points to the Google Search Scraper actor from Step 3.
5. Open the **Google Sheets** node ("Append row in sheet") → select your spreadsheet and sheet tab from the dropdowns, and confirm all 7 column mappings show as `{{ $json.fieldName }}` with no stray quote marks around them.

---

### Step 9 — Activate and test

1. Toggle the workflow to **Published / Active** (top right).
2. Message your bot on Telegram — try both a general question (`what is docker`) and a live one (`latest AI news today`).
3. Check your Google Sheet — a new logged row should appear for each message, with an IST timestamp.

---

## 🛠️ Common Issues

| Problem | Fix |
|---|---|
| Telegram Trigger fails with "An HTTPS URL must be provided for webhook" | You're accessing n8n via `localhost`. Set up ngrok (Step 6) and access n8n via the ngrok URL instead. |
| Bot suddenly stops replying, no error anywhere | ngrok and/or the n8n Docker container isn't running. Both must be active — check Docker Desktop and the ngrok terminal. |
| Google Sheets node: "Could not load list — 403 Forbidden — Google Drive API has not been used" | Enable the **Google Drive API** (not just Sheets API) in your Google Cloud project, wait ~1 minute, retry. |
| Google OAuth: "redirect_uri_mismatch" | The redirect URI in Google Cloud Console must match n8n's shown OAuth Redirect URL **exactly**, including the current ngrok domain. If your ngrok domain changed, update it in both places. |
| Telegram reply fails: "Bad Request: can't parse entities" | Gemini's raw markdown (`**bold**`) breaks Telegram's message parser. The workflow strips `*`, `_`, `` ` `` characters from the final response text before sending, and uses plain `•` characters for bullet formatting instead. |
| `Referenced node doesn't exist` in a Code node | A `$('Node Name')` reference doesn't exactly match the real node's name. Retype it using n8n's `$(` autocomplete dropdown instead of typing the name by hand. |
| Node shows "undefined" for all fields even though upstream data looks correct | The node is wired to the wrong parent (check the actual connection line on canvas), or it's using `$('SpecificNode')` when it should use `$json` because it receives input from multiple possible branches (e.g. the shared reply/logging nodes). |
| Gemini error: "The service is receiving too many requests from you" / quota exceeded | Free-tier rate limit hit from repeated testing. Wait 30–60 seconds and retry — this is exactly what the workflow's error-handling branches are designed to catch gracefully, replying to the user instead of failing silently. |
| Apify actor test returns results unrelated to your query | The query field wasn't in **expression mode** — click the `fx` icon next to the input field so `{{ }}` expressions actually get evaluated instead of sent as literal text. |
| Sheet columns show extra literal quote marks (e.g. `"NEWS"` instead of `NEWS`) | The expression was wrapped in escaped quotes, e.g. `="\"{{ $json.category }}\""`. Remove the extra `"` characters so it reads just `{{ $json.category }}`. |
| Telegram "typing..." indicator not visible | It only displays for ~5 seconds and disappears the instant the real reply arrives — on fast GENERAL answers the whole round-trip can finish before it's humanly noticeable. It's reliably visible on LIVE queries, which take longer end-to-end. |

---

## 📁 Project Structure

```
Live-Data-AI-Assistant/
├── README.md
├── Workflow/
│   └── workflow.json
├── Writeup/
│   └── Design_Report.pdf
├── Test-Report/
│   └── Test_Queries.pdf
├── Demo/
│   └── Live_Data_AI_Assistant.mp4
└── GoogleSheet/
    └── GoogleSheetLink.txt
```

---

## 👨‍💻 Author
**Sakthi Prasanth**
Project: Live Data AI Assistant — Telegram Bot 
---
powered by n8n, Apify, and Google Gemini
