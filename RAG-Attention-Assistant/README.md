# 🤖 RAG AI Assistant — "Attention Is All You Need" Telegram Bot

A Retrieval-Augmented Generation (RAG) assistant on Telegram that answers questions **strictly from the "Attention Is All You Need" paper** — never from the model's own general knowledge. It embeds the paper into Pinecone, retrieves the most relevant passages for every question, and replies with a cited, exactly-formatted answer, or an honest "not found" message when the paper genuinely doesn't cover the question.

---

## 🎥 Demo

[![Watch Demo Video](https://img.shields.io/badge/▶%20Watch%20Demo-Google%20Drive-2ec4a0?style=for-the-badge&logo=googledrive&logoColor=white)](https://drive.google.com/file/d/104K5kNfXG308EUPrW3dbKxPXr-kx2WRU/view?usp=sharing)

---

## 🎯 The Problem This Solves

A plain LLM chatbot will happily answer questions about "Attention Is All You Need" using whatever it already knows from training — which may be outdated, subtly wrong, or simply not grounded in the actual paper text. That's unacceptable for a knowledge-base assistant that needs to cite its source.

This project solves that by strictly grounding every answer in retrieved document content:

- The paper is **chunked, embedded, and stored in Pinecone** with metadata (document name, page number, chunk index) attached to every single chunk.
- Every Telegram question is **embedded and matched against Pinecone** to retrieve the most relevant passages before any answer is generated.
- The assistant is instructed — via a strict system prompt — to **only answer using retrieved context**, never its own knowledge, and to reply with an exact fallback sentence when nothing relevant is found.
- Every answer includes a **Sources** citation (document name, chunk number, page number) so the answer is always traceable back to the original text.
- **Conversation memory** (last 5 exchanges, per Telegram user) allows natural follow-up questions.
- Every interaction — success or failure — is **logged to Google Sheets**, and both realistic failure modes (no relevant content found vs. the AI service itself failing) are handled gracefully with distinct messages, so the bot never leaves the user with silence.

---

## 🏗️ Architecture & Flow

This project is **two separate n8n workflows**: one for ingesting the document, one for handling live queries.

### Workflow 1 — Document Ingestion

```
Manual Trigger
     │
     ▼
Download PDF  (fetches the paper from its public URL)
     │
     ▼
Extract Text From PDF
     │
     ▼
Chunk Text  (custom code: ~400 tokens/chunk, 50-token overlap,
             assigns id, document_name, page, chunk_index per chunk)
     │
     ▼
Pinecone Vector Store  (Insert mode)
     │         │
     │         ├── fed by: Default Data Loader (JSON mode, reads each
     │         │           chunk's text + attaches document_name/page/
     │         │           chunk_index as metadata; Text Splitting set
     │         │           to Custom + oversized Character Text Splitter,
     │         │           so chunks are NOT silently re-split)
     │         │
     │         └── fed by: Embeddings Google Gemini
     │
     ▼
Vectors stored in Pinecone (one namespace per document)
```

### Workflow 2 — Query (Telegram)

```
Telegram Trigger
     │
     ▼
Extract Question  (pulls chatId + question text)
     │
     ▼
AI Agent  ── strict system prompt: RAG-only, exact fallback sentence,
     │        exact "Answer:/Sources:" format, always search before answering
     │
     ├── ai_languageModel  → Google Gemini Chat Model
     ├── ai_memory          → Simple Memory (session key = chatId, last 5 turns)
     ├── ai_tool             → "Answer questions with a vector store"
     │                             │
     │                             ├── ai_vectorStore → Pinecone Vector Store (Retrieve)
     │                             ├── ai_embedding    → Embeddings Google Gemini
     │                             └── ai_languageModel → Google Gemini Chat Model
     │
     ├── SUCCESS output ──────────────────────┐
     │                                        ▼
     │                              Format Response
     │                     (extracts Sources/chunk IDs from the
     │                      answer text, sanitizes Markdown/LaTeX
     │                      characters that break Telegram, keeps
     │                      the tool's retrieval summary for logging)
     │                                        │
     └── ERROR output (Gemini/service failure)│
                    │                         │
                    ▼                         │
          Handle Agent Error                  │
     (distinct "service unavailable"          │
      message — NOT the same as the           │
      "not found in knowledge base" one)      │
                    │                         │
                    └───────────┬─────────────┘
                                ▼
                     Telegram - Send Answer
                                │
                                ▼
                   Google Sheets - Log Query
```

**Both the success path and the error path converge on the same Telegram-send and Sheets-logging nodes** — so the user always gets a reply, and every interaction is always logged, regardless of which branch actually fired.

---

## ✨ Key Design Decisions

- **Native n8n LangChain nodes over raw HTTP calls** — Pinecone Vector Store, Embeddings Google Gemini, and the Google Gemini Chat Model are all used as genuine built-in nodes (with real model/index dropdowns and simple credentials), rather than hand-built HTTP Request calls, wherever the assignment's requirements allowed it.
- **Custom chunking kept outside the Vector Store node** — n8n's Default Data Loader has its own internal text splitter that runs by default even on pre-chunked input, silently re-splitting our carefully sized 400-token chunks. This was diagnosed by comparing item counts (36 chunks in → 66 out) and fixed by switching Text Splitting to Custom with a deliberately oversized Character Text Splitter, so our own chunking is preserved untouched.
- **Document + page + chunk-level metadata, not just page-level** — achieved by doing chunking manually in code first, then feeding the pre-chunked JSON into Default Data Loader (rather than letting it parse and split the raw PDF itself), so `chunk_index` metadata survives all the way into Pinecone and back out into citations.
- **Two distinct failure messages, not one generic error** — "content not found in the knowledge base" (a successful run, just no relevant match) and "AI service temporarily unavailable" (an actual API/service failure) are deliberately different messages, routed through the AI Agent's separate success/error outputs, so a demo can clearly show both failure modes behaving correctly.
- **Telegram-safe text sanitization** — Gemini's raw answers can contain Markdown/LaTeX-style characters (`**bold**`, lone `*` bullets, `$d_k$` math notation, underscores) that break Telegram's message parser with a "can't parse entities" error. All asterisks, dollar signs, underscores, and backticks are stripped from the final answer text before sending, rather than relying on Telegram's Parse Mode setting.
- **Per-chat conversation memory** — Simple Memory's session key is explicitly set to the Telegram `chatId` (not the default chat-trigger-based session detection, which doesn't exist when using a Telegram Trigger), so each user gets their own independent last-5-turn memory.
- **Honest, documented trade-off on retrieval matching** — semantic search can miss a conceptually-correct chunk if the question's wording (e.g. "why did the authors **choose**...") doesn't overlap with the paper's own vocabulary (e.g. "**motivating** our use of self-attention"), even when the content genuinely exists in the index. This is a known limitation of embedding-based retrieval, mitigated (not eliminated) by increasing `topK` and prompting the Agent to retry with more academic phrasing before giving up.

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Chat interface | Telegram Bot API |
| Automation engine | n8n (self-hosted) |
| Vector database | Pinecone (serverless, cosine similarity) |
| AI / LLM | Google Gemini (embeddings + chat, free tier) |
| Orchestration | n8n's built-in AI Agent + LangChain-style vector store tool |
| Logging | Google Sheets (OAuth2) |
| Knowledge source | "Attention Is All You Need" (arXiv:1706.03762) |

---

## ⚙️ Setup & Installation

This is an **n8n workflow pair**, not a coded application — no repo to `git clone`. Set up the accounts below, run n8n, import both workflow files, and wire up your own credentials.

### Prerequisites
- A running n8n instance (self-hosted or cloud)
- A Telegram account
- A Google account (for Gemini + Sheets)
- A free [Pinecone](https://www.pinecone.io/) account

---

### Step 1 — Create your Telegram Bot
1. Open Telegram, search **`@BotFather`** → **Start**.
2. Send `/newbot`, give it a name and a username ending in `bot`.
3. Copy the **API Token** BotFather gives you.
4. Message your new bot once (search its username → **Start**) — required before it can message you first.

---

### Step 2 — Get a free Gemini API key
1. Go to [aistudio.google.com](https://aistudio.google.com/) → sign in.
2. **Get API key** → **Create API key**.
3. Copy the key (starts with `AIza...`).

---

### Step 3 — Create your Pinecone index
1. Sign up at [pinecone.io](https://www.pinecone.io/) (free tier).
2. **Create Index** → set **Dimension** to match your embedding model's output size (e.g. `3072` for `gemini-embedding-001` at default size), **Metric** to `cosine`, capacity mode **Serverless**.
3. Copy the index's **Host** URL and your **API Key** from the dashboard.

---

### Step 4 — Create your Google Sheet (for logging)
1. Create a blank spreadsheet, name it e.g. `RAG Assignment Logs`.
2. Rename the first tab to `Logs`.
3. Row 1 headers, one per column:
   ```
   timestamp | chatId | question | retrievedChunkIds | retrievedSummary | finalAnswer
   ```

---

### Step 5 — Add credentials inside n8n
Go to **Credentials → + Add Credential** for each:

- **Telegram API** — paste your Bot Token (Step 1)
- **Google Gemini(PaLM) Api** — paste your Gemini API key (Step 2), used by every Gemini-related node
- **Pinecone API** — paste your Pinecone API key (Step 3)
- **Google Sheets OAuth2 API** — connect and sign in with your Google account

---

### Step 6 — Import both workflows
1. **+ Create Workflow** → import `RAG_1_Document_Ingestion.json`.
2. Repeat for `RAG_2_Query_Workflow.json`.
3. In **every** node showing a credential dropdown, reselect the correct credential — credential IDs don't carry over between n8n instances.
4. In both **Pinecone Vector Store** nodes, reselect your real index from the dropdown and confirm the **Namespace** field.
5. In the ingestion workflow's **Default Data Loader** node, confirm **Text Splitting** is set to **Custom** with a **Character Text Splitter** (chunk size ~5000, overlap 0) attached — this is what prevents double-chunking.

---

### Step 7 — Run ingestion
1. Open the Ingestion workflow → **Test workflow**.
2. Check your Pinecone dashboard — the namespace should show one vector per chunk (roughly 30–40 for this paper), each with `document_name`, `page`, and `chunk_index` metadata populated.

---

### Step 8 — Activate and test the query workflow
1. Activate the Query workflow.
2. Message your bot a real question about the paper, and a completely unrelated question, to confirm both the cited-answer path and the fallback path work.
3. Check your Google Sheet — a new row should appear for each message.

---

## 🛠️ Common Issues

| Problem | Fix |
|---|---|
| `Error loading package ... ENOENT ... .tgz` when installing a community node | The needed node (Pinecone Vector Store, Embeddings Google Gemini) is already built into n8n by default — no install required. Just search for it when adding a node instead of installing a community package. |
| `No session ID found` on Simple Memory | It defaults to expecting a Chat Trigger node. Since a Telegram Trigger is used instead, set **Session ID → Define below**, and enter `{{ $('Extract Question').item.json.chatId }}`. |
| Pinecone namespace shows more records than expected (e.g. 66 instead of 36) | Default Data Loader's own internal text splitter is silently re-splitting already-chunked input. Set its **Text Splitting** to **Custom**, attach a **Character Text Splitter** with a chunk size far larger than your actual chunks (e.g. 5000), so it passes chunks through untouched. |
| Pinecone metadata shows generic `source: "blob"` instead of `document_name`/`page`/`chunk_index` | Default Data Loader's **Type of Data** reverted to Binary/PDF mode, or the **Metadata** fields under Options were empty/removed. Set Type of Data to **JSON**, Data to `{{ $json.text }}`, and add `document_name`/`page`/`chunk_index` metadata rows, each pointing to the matching `{{ $json.___ }}` field from the Chunk Text node. |
| Telegram send fails: `Bad Request: can't parse entities` | Gemini's raw answer contains Markdown/LaTeX characters (`**`, lone `*` bullets, `$...$`, `_`) that break Telegram's parser. Strip all of them in code before sending, rather than relying on the Parse Mode dropdown. |
| Bot goes completely silent instead of replying, on an API failure | The AI Agent node's **On Error** setting defaults to stopping the whole workflow. Set it to **Continue Using Error Output**, and route that error output to a small Code node that returns a friendly "service unavailable" message into the same Telegram/Sheets nodes used by the success path. |
| A conceptually correct question gets "I could not find relevant information" even though the paper covers it | Known limitation of embedding-based semantic search: the question's wording didn't overlap enough with the paper's own vocabulary to rank highly. Mitigate by increasing `topK` on the retrieval node, and adding a system-prompt instruction encouraging the Agent to retry with more academic phrasing before concluding nothing was found. |
| "Powered by n8n" footer appears under every Telegram reply | Turn off **Append n8n Attribution** in the Telegram send node's Additional Fields. |

---

## 📁 Project Structure

```
RAG-Attention-Assistant/
├── README.md
├── Workflows/
│   ├── RAG_1_Document_Ingestion.json
│   └── RAG_2_Query_Workflow.json
├── Writeup/
│   └── Design_Report.pdf
├── Test-Report/
│   └── Test_Queries.pdf
├── Demo/
│   └── RAG_Attention_Assistant_Demo.mp4
└── GoogleSheet/
    └── GoogleSheetLink.txt
```

---

## 👨‍💻 Author
**Sakthi Prasanth**

Project: RAG AI Assistant — "Attention Is All You Need" Telegram Bot

Built with n8n, Pinecone, and Google Gemini
