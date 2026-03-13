# One Step After — AI Chat System: Handover Guide

## Table of Contents

1. System Overview
2. Architecture & How It All Fits Together
3. External Services & Accounts
4. Workflow 1: Document Ingestion
5. Workflow 2: Chat API (Webhook)
6. Workflow 3: Chat Widget (Webchat)
7. The Knowledge Base (Zilliz)
8. Adding, Updating, and Removing Documents
9. Tuning & Configuration
10. The System Prompt
11. Costs & Usage Monitoring
12. Troubleshooting
13. Common Maintenance Tasks
14. Credential Locations
15. Security Considerations

---

## 1. System Overview

The One Step After AI chat system answers user questions about estate planning, probate, powers of attorney, and related topics using your own documentation as the primary knowledge source. When a user asks a question, the system searches your document library, finds the most relevant passages, and uses Claude (Anthropic's AI) to synthesize a clear, cited answer.

There are three n8n Cloud workflows that make up the system:

| Workflow | Purpose | Trigger |
|----------|---------|---------|
| **RAG Ingestion** | Reads documents from Google Drive, chunks them, generates vector embeddings, and stores them in the Zilliz database | Scheduled weekly (Mondays at 2:00 PM) or manual execution |
| **RAG Chat — Webhook** | The API that the Bubble app calls. Accepts a question, retrieves relevant docs, and returns a structured JSON response | HTTP POST to webhook URL |
| **RAG Chat — Webchat** | A standalone chat widget that can be embedded on any webpage with a JavaScript snippet | User interaction with the chat widget |

---

## 2. Architecture & How It All Fits Together

```
                    DOCUMENT INGESTION
                    ==================

Google Drive (Shared Drive)
  └─ Subfolders containing docs
       │
       ▼
  n8n Ingestion Workflow
       │
       ├─ Export text from Google Docs (native API)
       ├─ Convert DOCX/XLSX/PDF → Google format → export text
       ├─ Split text into chunks (~1500 chars each)
       ├─ Generate vector embeddings (Voyage AI)
       └─ Store chunks + vectors in Zilliz Cloud
                         │
                         ▼
               Zilliz Cloud Database
              (knowledge_base collection)
              50+ chunks across 20+ documents


                      CHAT FLOW
                      =========

User asks a question
       │
       ▼
n8n Chat Workflow (webhook or webchat)
       │
       ├─ Embed the question (Voyage AI)
       ├─ Search Zilliz for the 5 most similar chunks
       │
       ├─ If good matches found (score > 0.35):
       │    └─ Build context from matched chunks
       │
       ├─ If no good matches AND question is on-topic:
       │    └─ Web search via Brave API
       │
       ├─ If question is off-topic:
       │    └─ Redirect (no search performed)
       │
       └─ Send context + question to Claude → response
```

### Key Concept: Vector Embeddings

The system converts both documents and questions into "vector embeddings" — lists of 1024 numbers that represent the meaning of the text. When a user asks a question, the system finds document chunks whose embeddings are most similar to the question's embedding (using cosine similarity). This is what makes it possible to find relevant information even when the user doesn't use the exact same words as the document.

---

## 3. External Services & Accounts

| Service | What It Does | Account/Dashboard |
|---------|-------------|-------------------|
| **n8n Cloud** | Hosts and runs all three workflows | https://onestepafter.app.n8n.cloud |
| **Google Drive** | Source document storage (Shared Drive) | Folder ID: `<GDRIVE_FOLDER_ID — shared separately>` |
| **Voyage AI** | Generates vector embeddings for text | https://dash.voyageai.com |
| **Zilliz Cloud** | Vector database that stores and searches document chunks | https://cloud.zilliz.com |
| **Anthropic** | Claude AI model that generates answers | https://console.anthropic.com |
| **Brave Search** | Web search fallback when docs don't have the answer | https://api.search.brave.com |

### API Keys in Use

All API keys are hardcoded directly in the n8n workflow Code nodes (not in environment variables, because n8n Cloud doesn't support `$env` in Code nodes on all plans). See the Credential Locations section at the end of this document for exactly where each key lives.

---

## 4. Workflow 1: Document Ingestion

**Name in n8n:** `RAG Ingestion — Google Drive → Zilliz (Cloud-Compatible)`
**Nodes:** 24
**Purpose:** Reads documents from Google Drive, converts them to text, splits into searchable chunks, and loads them into the vector database.

### How to Run It

There are two ways documents get ingested:

**Scheduled (weekly):** The workflow runs automatically every Monday at 2:00 PM. It scans the entire Google Drive shared folder, processes every supported document, and loads/refreshes them all in Zilliz. This catches any new files, updates, or changes made during the week. The workflow must be toggled to "Active" in n8n for the schedule to run.

**Manual (on-demand):** Click "Execute Workflow" in n8n at any time. This performs the same full scan and is useful when you've just added documents and don't want to wait until Monday.

### Supported File Types

| Type | How It's Processed |
|------|-------------------|
| Google Docs | Exported directly as plain text via the Drive API |
| DOCX (Word) | Converted to a Google Doc first, then exported as text |
| XLSX (Excel) | Converted to a Google Sheet, then exported as CSV text |
| PDF | Converted to a Google Doc (OCR-dependent), then exported as text |
| Google Drive Shortcuts | Resolved to their target file, then processed by type |

Files that don't match these types (images, videos, folders) are silently skipped.

### Node-by-Node Walkthrough

```
Manual Trigger ──────────┐
                         ├─→ Search files and folders2
Schedule Trigger ────────┘     (Mondays at 2:00 PM)
  └─ Search files and folders2
       Finds all subfolders in the Shared Drive root folder
       └─ Search files and folders3
            Searches each subfolder for supported file types
            (Google Docs, DOCX, XLSX, PDF, Shortcuts)
            └─ Supported File Types?
                 Filters to only supported mimeTypes
                 └─ Resolve Shortcuts
                      Resolves shortcut targets, tags each file with
                      routing info (_exportType: direct or convert)
                      └─ Process One Doc at a Time (Loop)
                           Processes one document per iteration to
                           avoid memory limits
                           │
                           ├─ Process or Skip?
                           │    Skips unsupported shortcut targets
                           │
                           ├─ Direct or Convert?
                           │    ├─ [direct] Export Doc as Text
                           │    │    Google Docs: GET /export?mimeType=text/plain
                           │    │
                           │    └─ [convert] Copy to Google Format
                           │         POST /files/{id}/copy (converts DOCX→Doc, etc.)
                           │         └─ Copy Succeeded?
                           │              ├─ [yes] Export Converted Doc
                           │              │    └─ Delete Temp Copy (cleanup)
                           │              └─ [no] Conversion Failures (Log)
                           │
                           ├─ Extract Text & Metadata
                           │    Pulls docId, docTitle, lastModified, textContent
                           │
                           ├─ Chunk Text + Attach Metadata
                           │    Splits text into ~1500 char chunks with 400 char overlap
                           │    Adds: id, applicableStates, docType
                           │
                           ├─ Delete Old Chunks (Zilliz)
                           │    Removes previous chunks for this docId
                           │
                           ├─ Embed Chunks (Voyage AI)
                           │    Calls Voyage AI to get 1024-dim vectors
                           │    Batches of 8, with individual retry on failure
                           │
                           ├─ Upsert into Zilliz
                           │    Inserts chunks into knowledge_base collection
                           │
                           └─ Log Ingestion Complete → loops back
```

### Error Handling

The ingestion pipeline is designed to be resilient — a single failed document won't stop the rest:

- **Export Doc as Text** and **Export Converted Doc** have `retryOnFail` enabled (retries 3 times on Google API errors like 500s)
- **Copy to Google Format** has `continueOnFail` — if conversion fails (e.g., file too large), it logs the failure and the loop continues
- **Embed Chunks** tries batches of 8 first, then falls back to embedding one chunk at a time if a batch fails
- **All failure/skip paths loop back** to the "Process One Doc at a Time" node so the next document always gets processed

### Chunking Configuration

These values are in the "Chunk Text + Attach Metadata" node:

| Setting | Value | What It Means |
|---------|-------|---------------|
| `CHUNK_SIZE` | 1500 chars | Target size for each chunk (~375 tokens) |
| `CHUNK_OVERLAP` | 400 chars | Overlap between consecutive chunks to preserve context |
| `MAX_CHARS` | 60000 chars | Hard limit per chunk (Zilliz VarChar max is 65535) |

The chunking is paragraph-aware — it splits on `\n\n` boundaries when possible. For documents without paragraph breaks (or with very long paragraphs), it falls back to sentence/word-boundary splitting.

### Metadata Extraction

Each chunk automatically gets:

- **applicableStates** — US state codes extracted from the document title (e.g., a doc titled "CO Title 15..." gets tagged with `["CO"]`). Docs without state references get `["ALL"]`.
- **docType** — Classified from the title: `policy`, `faq`, `guide`, `legal`, `pricing`, or `general`.

---

## 5. Workflow 2: Chat API (Webhook)

**Name in n8n:** `RAG Chat — Webhook → Retrieve → Claude`
**Nodes:** 12
**Purpose:** The primary chat API that the Bubble application calls. Returns structured JSON responses.

### Endpoint

```
POST https://onestepafter.app.n8n.cloud/webhook/<WEBHOOK_PATH — shared separately>
```

### Authentication

Every request must include an `x-api-key` header:
```
x-api-key: <API_KEY — shared separately>
```

Requests without a valid key receive a 401 response. The key is checked in the "Parse & Validate Request" Code node.

### Request Format

```json
{
  "message": "What are the probate thresholds in Colorado?",
  "sessionId": "user_abc123_session_001",
  "conversationHistory": [
    { "role": "user", "content": "Previous question" },
    { "role": "assistant", "content": "Previous answer" }
  ],
  "metadata": {
    "userName": "Benji",
    "userState": "CO",
    "accountId": "bubble_user_id",
    "plan": "premium"
  }
}
```

### Response Format

```json
{
  "response": "Based on our documentation, Colorado's probate thresholds are...",
  "contextSource": "internal_docs",
  "sessionId": "user_abc123_session_001",
  "sourceDocs": [
    { "title": "US State Probate Thresholds.xlsx", "docId": "1lLj..." }
  ],
  "usage": { "input_tokens": 1842, "output_tokens": 356 }
}
```

### Context Source Values

| Value | Meaning |
|-------|---------|
| `internal_docs` | Answer is grounded in One Step After's knowledge base |
| `web_search` | On-topic question but not covered in docs; supplemented with web results |
| `off_topic` | User asked something unrelated; response redirects them |
| `error` | Something went wrong server-side |

### Conversation History

Conversation history is managed **client-side** (in Bubble). The API is stateless. The client sends the previous exchanges in `conversationHistory`, and the workflow passes them to Claude so it understands follow-up questions like "What about California?" after a question about Colorado.

The workflow trims history to the last 10 exchanges to keep within token limits.

### Node-by-Node Walkthrough

```
Chat Webhook (POST)
  └─ Parse & Validate Request
       Checks API key, validates message, extracts metadata
       └─ Has Error?
            ├─ [yes] Return Error (401 or 400)
            └─ [no] Embed Query (Voyage AI)
                 Converts the question to a 1024-dim vector
                 └─ Search Zilliz (Vector Search)
                      Finds top 5 most similar chunks (cosine similarity)
                      Applies state filter if userState is set
                      └─ Docs Relevant? (score > 0.35?)
                           ├─ [yes] Build Doc Context
                           │    Formats matched chunks with source titles
                           │
                           └─ [no] Web Search Fallback
                                ├─ On-topic? → Brave search (scoped to estate/probate)
                                └─ Off-topic? → Redirect context (no search)
                                │
                           Merge Context
                                └─ Call Claude API
                                     Sends system prompt + context + history + question
                                     └─ Return Response (JSON)
```

### State-Aware Filtering

When `metadata.userState` is provided (e.g., `"CO"`), the Zilliz search includes a filter:
```
applicableStates like "%CO%" or applicableStates like "%ALL%"
```
This prioritizes state-specific documents while still including nationally applicable ones.

---

## 6. Workflow 3: Chat Widget (Webchat)

**Name in n8n:** `RAG Chat — Webchat Widget — Retrieve → Claude`
**Nodes:** 9
**Purpose:** A standalone chat interface that can be embedded on any webpage using n8n's official `@n8n/chat` JavaScript package.

### How It Works

This workflow uses n8n's built-in **Chat Trigger** node instead of a raw webhook. The Chat Trigger provides a hosted chat interface that the `@n8n/chat` npm package connects to. The user sees a floating chat button on the page, clicks it to open a chat window, and messages flow through the same Voyage AI → Zilliz → Claude pipeline.

### Embedding on a Website

Add this before the closing `</body>` tag (replace the `webhookUrl` with the production URL from the Chat Trigger node):

```html
<link href="https://cdn.jsdelivr.net/npm/@n8n/chat/dist/style.css" rel="stylesheet" />
<script type="module">
  import { createChat } from 'https://cdn.jsdelivr.net/npm/@n8n/chat/dist/chat.bundle.es.js';

  createChat({
    webhookUrl: 'https://onestepafter.app.n8n.cloud/webhook/XXXXXXXX/chat',
    mode: 'window',
    chatInputKey: 'chatInput',
    metadata: {},
    showWelcomeScreen: true,
    initialMessages: [
      'Hi! I\'m here to help with estate planning, probate, and the steps involved after someone passes away.',
      'You can ask me about wills, trusts, powers of attorney, probate thresholds, executor responsibilities, and more.'
    ],
    i18n: {
      en: {
        title: 'One Step After',
        subtitle: 'Estate planning & execution assistant',
        getStarted: 'Start a Conversation',
        inputPlaceholder: 'Ask about estate planning, probate, wills...',
        footer: ''
      }
    }
  });
</script>
```

This uses the official `@n8n/chat` package from npm (https://www.npmjs.com/package/@n8n/chat), loaded via the jsDelivr CDN. No build step is needed.

### Differences from the Webhook Version

| Feature | Webhook API | Webchat Widget |
|---------|-------------|----------------|
| Frontend | Bubble builds the UI | Pre-built n8n widget |
| Conversation history | Client-managed, sent per request | Per-session only |
| Authentication | API key header | None (use allowedOrigins to restrict) |
| State filtering | Via metadata.userState | Not available (defaults to ALL) |
| User metadata | Full control | Limited |
| Response format | Structured JSON | Rendered Markdown in chat bubble |

---

## 7. The Knowledge Base (Zilliz)

### Collection: `knowledge_base`

| Field | Type | Description |
|-------|------|-------------|
| `id` | VarChar (PK) | Unique chunk ID: `{docId}_chunk_{index}` |
| `vector` | FloatVector (1024) | Voyage AI embedding |
| `text` | VarChar (65535) | The actual chunk text |
| `docId` | VarChar | Google Drive file ID |
| `docTitle` | VarChar | Document title |
| `lastModified` | VarChar | ISO timestamp |
| `applicableStates` | VarChar | JSON array of state codes, e.g., `["CO"]` or `["ALL"]` |
| `docType` | VarChar | Category: general, policy, faq, guide, legal, pricing |
| `chunkIndex` | Int64 | Position within the document |

### Connection Details

- **Endpoint:** `<ZILLIZ_ENDPOINT — shared separately>`
- **Cluster type:** Serverless (free tier, 200MB storage)
- **Index:** AUTOINDEX on `vector` field, COSINE metric
- **Dashboard:** https://cloud.zilliz.com

### Querying the Database

Check how many chunks are stored (note: stats may lag behind on serverless):
```bash
curl -X POST "<ZILLIZ_ENDPOINT — shared separately>/v2/vectordb/entities/query" \
  -H "Authorization: Bearer YOUR_ZILLIZ_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"collectionName":"knowledge_base","filter":"chunkIndex >= 0","outputFields":["docTitle","chunkIndex"],"limit":100}'
```

---

## 8. Adding, Updating, and Removing Documents

### Adding a New Document

1. Add the document to any subfolder within the Shared Drive root folder (`<GDRIVE_FOLDER_ID — shared separately>`)
2. Supported formats: Google Doc, DOCX, XLSX, PDF
3. The document will be automatically picked up on the next scheduled run (Mondays at 2:00 PM)
4. If you need it ingested sooner, run the ingestion workflow manually by clicking "Execute Workflow" in n8n

### Updating an Existing Document

1. Edit the document in Google Drive
2. The changes will be picked up on the next scheduled run (Mondays at 2:00 PM), or run the workflow manually
3. The workflow automatically deletes old chunks for that document before re-inserting, so you won't get duplicates

### Removing a Document

1. Delete or move the document out of the Shared Drive folder
2. The chunks will remain in Zilliz until you either:
   - Drop and recreate the collection (see below)
   - Manually delete chunks via the Zilliz API:
     ```bash
     curl -X POST "ZILLIZ_ENDPOINT/v2/vectordb/entities/delete" \
       -H "Authorization: Bearer ZILLIZ_API_KEY" \
       -H "Content-Type: application/json" \
       -d '{"collectionName":"knowledge_base","filter":"docId == \"THE_GOOGLE_DRIVE_FILE_ID\""}'
     ```

### Full Re-Ingestion (Nuclear Option)

If you want to start fresh:

1. Drop the collection:
   ```bash
   curl -X POST "ZILLIZ_ENDPOINT/v2/vectordb/collections/drop" \
     -H "Authorization: Bearer ZILLIZ_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"collectionName":"knowledge_base"}'
   ```

2. Recreate the collection:
   ```bash
   curl -X POST "ZILLIZ_ENDPOINT/v2/vectordb/collections/create" \
     -H "Authorization: Bearer ZILLIZ_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"collectionName":"knowledge_base","schema":{"autoId":false,"enableDynamicField":true,"fields":[{"fieldName":"id","dataType":"VarChar","isPrimary":true,"elementTypeParams":{"max_length":"256"}},{"fieldName":"vector","dataType":"FloatVector","elementTypeParams":{"dim":"1024"}},{"fieldName":"text","dataType":"VarChar","elementTypeParams":{"max_length":"65535"}},{"fieldName":"docId","dataType":"VarChar","elementTypeParams":{"max_length":"256"}},{"fieldName":"docTitle","dataType":"VarChar","elementTypeParams":{"max_length":"512"}},{"fieldName":"lastModified","dataType":"VarChar","elementTypeParams":{"max_length":"64"}},{"fieldName":"applicableStates","dataType":"VarChar","elementTypeParams":{"max_length":"2048"}},{"fieldName":"docType","dataType":"VarChar","elementTypeParams":{"max_length":"64"}},{"fieldName":"chunkIndex","dataType":"Int64"}]},"indexParams":[{"fieldName":"vector","indexName":"vector_idx","metricType":"COSINE","indexType":"AUTOINDEX"}]}'
   ```

3. Run the ingestion workflow manually

---

## 9. Tuning & Configuration

### Relevance Threshold

In both chat workflows, the "Search Zilliz (Vector Search)" node has:
```javascript
const RELEVANCE_THRESHOLD = 0.35;
```

This controls when the system uses your documents vs. falls back to web search. Cosine similarity scores range from 0 to 1:

| Score Range | Meaning | Example |
|-------------|---------|---------|
| 0.60 – 1.00 | Excellent match | "Probate thresholds in Colorado" → Probate Thresholds doc |
| 0.40 – 0.60 | Good match | "What happens after someone dies" → multiple docs |
| 0.35 – 0.40 | Marginal match | Somewhat related |
| Below 0.35 | Poor match | Falls back to web search |

**To make the system rely more on your docs:** Lower the threshold (e.g., 0.25). The risk is less relevant chunks being used.

**To make the system more selective:** Raise the threshold (e.g., 0.50). The risk is more questions falling through to web search.

### Number of Results

In the search node, `limit: 5` controls how many chunks are retrieved. More chunks = more context for Claude but higher token costs. 3–5 is the sweet spot for most use cases.

### Claude Model

The system uses `claude-sonnet-4-5-20250929`. To change models, update the `model` field in the "Call Claude API" node in both chat workflows. Options include:

- `claude-sonnet-4-5-20250929` — Current model. Good balance of quality and cost.
- A newer model version — Check https://docs.anthropic.com/en/docs/about-claude/models for the latest.

### Max Tokens

Currently set to `1024` in the Claude API call. Increase for longer responses (up to 4096 for most models). Higher values cost more per response.

---

## 10. The System Prompt

The system prompt is in the "Call Claude API" Code node in both chat workflows. It defines the assistant's personality, scope, and behavior rules. Here's what it covers:

- **Identity:** "Knowledgeable and compassionate assistant for One Step After"
- **Scope:** Estate planning, probate, wills, trusts, POA, death notification, asset management, etc.
- **Guidelines:** Cite sources, be state-specific, be compassionate, don't make up legal info, not a lawyer
- **Off-topic handling:** Politely redirect, don't answer, don't web search

To modify the assistant's behavior, edit the `systemPrompt` string in the "Call Claude API" node. Common changes:

- **Add new topic areas** — Update the "YOUR SCOPE" list
- **Change tone** — Modify the "GUIDELINES" section
- **Add company-specific info** — Add details about pricing, plans, or features
- **Adjust off-topic handling** — Make it stricter or more lenient

Remember to update the prompt in **both** chat workflows (webhook and webchat) to keep them consistent.

---

## 11. Costs & Usage Monitoring

### Per-Request Costs (Approximate)

| Service | Per Chat Message | Notes |
|---------|-----------------|-------|
| Voyage AI (embedding) | ~$0.0001 | 1 query embedding per message |
| Zilliz Cloud | Free | Serverless free tier (200MB storage) |
| Anthropic (Claude) | ~$0.003–0.01 | Depends on context size and response length |
| Brave Search | ~$0.005 | Only when docs aren't relevant (fallback) |

The `usage` field in webhook responses shows exact token counts. Monitor these to track costs.

### Ingestion Costs

| Service | Per Document | Notes |
|---------|-------------|-------|
| Voyage AI | ~$0.001–0.01 | Depends on document size (more chunks = more embeddings) |
| Zilliz Cloud | Free | Within the 200MB free tier for current doc volume |

### Free Tier Limits to Watch

- **Zilliz Serverless:** 200MB storage. Current usage is well under this.
- **Voyage AI:** Rate limited at ~300 requests/minute on the free tier.
- **Brave Search:** Free tier has monthly query limits.

---

## 12. Troubleshooting

### "No results" or Poor Answers

1. Check that the ingestion workflow ran successfully (look at execution history in n8n)
2. Verify data exists in Zilliz by running the query command in Section 7
3. Try lowering the `RELEVANCE_THRESHOLD` in the search node
4. Check if the document was properly chunked — very short docs may produce only 1 chunk

### Chat Webhook Returns 401

The API key in the request doesn't match. Check the `x-api-key` header matches the key in the "Parse & Validate Request" node.

### Chat Webhook Returns 500 or Times Out

Check the n8n execution log for the specific error. Common causes:
- Voyage AI rate limit (429) — wait and retry
- Anthropic API error — check API key and account status
- Zilliz connection issue — check endpoint URL and API key

### Ingestion Fails for a Specific Document

Check the execution history in n8n and look at which node failed:
- **Export Doc as Text** (500 error) — Google Drive API hiccup; usually resolves on retry
- **Copy to Google Format** (413 error) — File too large for conversion; split the source file into smaller parts
- **Embed Chunks** (400 error) — Usually a chunk with empty text; the retry-per-chunk fallback should handle this
- **Upsert into Zilliz** (1100 error) — A chunk exceeds 65535 characters; lower `MAX_CHARS` in the chunk node

### Webchat Widget Doesn't Appear

1. Verify the workflow is toggled to **Active** in n8n
2. Check the `webhookUrl` in the JavaScript snippet matches the Chat Trigger's production URL
3. Check the browser console for JavaScript errors
4. Verify `allowedOrigins` in the Chat Trigger node includes the website's domain

---

## 13. Common Maintenance Tasks

### Rotate the Chat API Key

1. Open the webhook chat workflow
2. Edit the "Parse & Validate Request" Code node
3. Replace the `API_KEY` string with a new value
4. Save the workflow
5. Give the new key to the Bubble developer

### Update Voyage AI or Anthropic API Keys

Search for the old key across all Code nodes in all three workflows. The keys appear in:
- Ingestion: "Embed Chunks (Voyage AI)" node
- Webhook Chat: "Embed Query (Voyage AI)" and "Call Claude API" nodes
- Webchat: "Embed Query (Voyage AI)" and "Call Claude API" nodes

### Change the Google Drive Source Folder

1. Open the ingestion workflow
2. Edit "Search files and folders2" — change the folder URL/ID

### Restrict Webchat to Specific Domains

1. Open the webchat workflow
2. Click on the "Chat Trigger" node
3. Change `allowedOrigins` from `*` to your domains: `onestepafter.com,*.onestepafter.com`

---

## 14. Credential Locations

Every API key is hardcoded in Code nodes (n8n Cloud doesn't reliably support `$env` in Code nodes). Here's exactly where each one lives:

### Ingestion Workflow

| Key | Node | Variable Name |
|-----|------|--------------|
| Voyage AI | Embed Chunks (Voyage AI) | `VOYAGE_API_KEY` |
| Zilliz Endpoint | Delete Old Chunks (Zilliz REST)1 | `ZILLIZ_ENDPOINT` |
| Zilliz Endpoint | Upsert into Zilliz (REST)1 | `ZILLIZ_ENDPOINT` |
| Zilliz API Key | Delete Old Chunks (Zilliz REST)1 | `ZILLIZ_API_KEY` |
| Zilliz API Key | Upsert into Zilliz (REST)1 | `ZILLIZ_API_KEY` |
| Google Drive OAuth2 | Search, Export, and Copy nodes | Via n8n credential (not in code) |

### Webhook Chat Workflow

| Key | Node | Variable Name |
|-----|------|--------------|
| Chat API Key | Parse & Validate Request | `API_KEY` |
| Voyage AI | Embed Query (Voyage AI) | `VOYAGE_API_KEY` |
| Zilliz Endpoint | Search Zilliz (Vector Search) | `ZILLIZ_ENDPOINT` |
| Zilliz API Key | Search Zilliz (Vector Search) | `ZILLIZ_API_KEY` |
| Brave Search | Web Search Fallback | `BRAVE_API_KEY` |
| Anthropic | Call Claude API | `ANTHROPIC_API_KEY` |

### Webchat Workflow

| Key | Node | Variable Name |
|-----|------|--------------|
| Voyage AI | Embed Query (Voyage AI) | `VOYAGE_API_KEY` |
| Zilliz Endpoint | Search Zilliz (Vector Search) | `ZILLIZ_ENDPOINT` |
| Zilliz API Key | Search Zilliz (Vector Search) | `ZILLIZ_API_KEY` |
| Brave Search | Web Search Fallback | `BRAVE_API_KEY` |
| Anthropic | Call Claude API | `ANTHROPIC_API_KEY` |

---

## 15. Security Considerations

- **API keys in code:** All keys are embedded in n8n Code nodes. Anyone with access to the n8n workspace can see them. Limit n8n access to trusted team members only.
- **Webhook API key:** The `osa_` key protects the webhook chat endpoint. Rotate it periodically and if you suspect exposure.
- **Webchat widget:** Use `allowedOrigins` to restrict which domains can embed it.
- **Zilliz API key:** Provides full read/write access to the vector database. If compromised, rotate it in the Zilliz dashboard and update all three workflows.
- **Google Drive OAuth2:** Managed through n8n's credential system (encrypted at rest). The OAuth token has access to the connected Google account's Drive.
- **CORS:** Both chat workflows currently allow all origins (`*`). For production, restricted to your specific domains.
- **Data in transit:** All API calls use HTTPS. Data at rest in Zilliz is encrypted by the provider.

---

*Last updated: March 2026*
*System built on: n8n Cloud, Voyage AI (voyage-3.5), Zilliz Cloud (Serverless), Anthropic Claude (claude-sonnet-4.5), Brave Search API*
