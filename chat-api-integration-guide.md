# One Step After — Chat API Integration Guide

## Overview

This document covers everything needed to integrate the One Step After AI chat assistant into the Bubble application. The chat API is a webhook hosted on n8n Cloud that accepts a user's question, retrieves relevant information from our estate planning knowledge base, and returns a contextual answer powered by Claude (Anthropic).

---

## API Endpoint

**Production URL:**
```
POST https://onestepafter.app.n8n.cloud/webhook/chat
```

**Content-Type:** `application/json`

No API key or authentication header is required — the webhook is open. CORS is set to allow all origins (`*`). If we want to lock this down later, we can restrict to specific domains in the n8n workflow.

---

## Request Format

```json
{
  "message": "What are the probate thresholds in Colorado?",
  "sessionId": "user_abc123_1709571600",
  "conversationHistory": [
    {
      "role": "user",
      "content": "What is a power of attorney?"
    },
    {
      "role": "assistant",
      "content": "A power of attorney (POA) is a legal document that..."
    }
  ],
  "metadata": {
    "userName": "Benji",
    "userState": "CO",
    "accountId": "bubble_user_id_here",
    "plan": "premium"
  }
}
```

### Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message` | string | **Yes** | The user's current question. Must be non-empty. |
| `sessionId` | string | No | A unique identifier for this chat session. Use the Bubble user ID + a timestamp or random string. Falls back to `"anonymous"` if omitted. |
| `conversationHistory` | array | No | Previous messages in this conversation (see Conversation History section below). |
| `metadata.userName` | string | No | The user's display name. Defaults to `"User"`. |
| `metadata.userState` | string | No | Two-letter US state code (e.g., `"CO"`, `"CA"`, `"NY"`). Used to filter and prioritize state-specific legal information. Defaults to `"ALL"`. |
| `metadata.accountId` | string | No | The user's Bubble account ID. For logging/analytics. |
| `metadata.plan` | string | No | The user's subscription plan. Can be used for rate limiting or feature gating in the future. |

---

## Response Format

### Success (HTTP 200)

```json
{
  "response": "Based on our documentation, Colorado's probate thresholds are...",
  "contextSource": "internal_docs",
  "sessionId": "user_abc123_1709571600",
  "sourceDocs": [
    {
      "title": "US State Probate Thresholds.xlsx",
      "docId": "1lLj62fHdJm8Iuo-SaCUBV_JjvApRNHhP"
    },
    {
      "title": "CO Title 15 Trusts, Estates, Fiduciaries [TEXT] [Part 1]",
      "docId": "1kO2Cs25Csnvb_Gk_Hhp3fD-BryUSFoUDYtYy_uLkSFk"
    }
  ],
  "usage": {
    "input_tokens": 1842,
    "output_tokens": 356
  }
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `response` | string | The AI-generated answer. May contain Markdown formatting (bold, headers, lists). |
| `contextSource` | string | Where the answer came from: `"internal_docs"`, `"web_search"`, `"off_topic"`, or `"error"`. |
| `sessionId` | string | Echoed back from the request. |
| `sourceDocs` | array | Documents referenced in the answer (only when `contextSource` is `"internal_docs"`). Each has `title` and `docId`. |
| `usage` | object | Token usage from Claude. Useful for monitoring costs. |

### Error (HTTP 400)

```json
{
  "error": "Message is required",
  "status": 400
}
```

### Context Source Values

| Value | Meaning | UI Suggestion |
|-------|---------|---------------|
| `internal_docs` | Answer is grounded in One Step After's knowledge base. | Show source badges. High confidence. |
| `web_search` | Question was on-topic but not covered in our docs. Supplemented with web results. | Show a note like "Based on general information." |
| `off_topic` | User asked something unrelated to estate planning. The response will redirect them. | No special treatment needed — the response text handles the redirect. |
| `error` | Something went wrong server-side. | Show a retry button or generic error message. |

---

## Conversation History

Conversation history is managed **client-side** (in Bubble). The API is stateless — it doesn't store any conversation data between requests.

### How It Works

1. User sends their first message → `conversationHistory` is empty or omitted
2. API returns a response
3. Bubble appends both the user message and assistant response to a local list
4. On the next message, Bubble sends the accumulated history in `conversationHistory`

### Bubble Implementation

Store the conversation history as a **list of JSON objects** on the page or in a custom state. After each API call:

```
// Pseudocode for Bubble workflow:

// 1. Before API call, build the conversationHistory from your stored messages
// 2. Send the API request with the current message + history
// 3. After receiving the response:

Append to chat_messages list:
  { "role": "user", "content": [the message the user just typed] }
  { "role": "assistant", "content": [response from API] }

// 4. Next API call includes the updated chat_messages as conversationHistory
```

### Rules

- The API trims history to the **last 10 message pairs** (20 items), so you don't need to manage trimming on the client side.
- Each history item must have `role` (either `"user"` or `"assistant"`) and `content` (the message text).
- Only include the `response` field from the API in the assistant content — don't include `sourceDocs` or other metadata in the history.
- For a "New Chat" button, simply clear the stored conversation history and generate a new `sessionId`.

### Example Flow

**Turn 1 — User asks their first question:**
```json
{
  "message": "What are the probate thresholds in Colorado?",
  "sessionId": "session_001",
  "conversationHistory": [],
  "metadata": { "userName": "Benji", "userState": "CO" }
}
```

**Turn 2 — Follow-up question with history:**
```json
{
  "message": "What about California?",
  "sessionId": "session_001",
  "conversationHistory": [
    { "role": "user", "content": "What are the probate thresholds in Colorado?" },
    { "role": "assistant", "content": "Based on our documentation, Colorado's probate thresholds are $74,000 or less for personal property..." }
  ],
  "metadata": { "userName": "Benji", "userState": "CA" }
}
```

The AI understands "What about California?" refers to probate thresholds because it sees the prior exchange.

---

## Bubble-Specific Implementation Notes

### Setting Up the API Connector

1. Go to **Plugins → API Connector**
2. Add a new API called `OneStepAfterChat`
3. Configure the call:
   - **Name:** `Send Message`
   - **Method:** `POST`
   - **URL:** `https://onestepafter.app.n8n.cloud/webhook/chat`
   - **Headers:**
     - `Content-Type`: `application/json`
   - **Body type:** JSON
   - **Body (use as template, mark fields as dynamic):**

```json
{
  "message": "<message>",
  "sessionId": "<sessionId>",
  "conversationHistory": <conversationHistory>,
  "metadata": {
    "userName": "<userName>",
    "userState": "<userState>",
    "accountId": "<accountId>",
    "plan": "<plan>"
  }
}
```

4. Mark `message`, `sessionId`, `conversationHistory`, `userName`, `userState`, `accountId`, and `plan` as dynamic values
5. Initialize the call with test data to let Bubble detect the response structure

### Data Types

Create a custom data type or use Bubble's built-in state to track chat:

**Option A — Page State (simpler, no database writes):**
- Custom state `chat_messages` (type: text, list) — store each message as a JSON string
- Custom state `session_id` (type: text)

**Option B — Database (persistent history):**
- Data type: `ChatMessage`
  - `session_id` (text)
  - `role` (text: "user" or "assistant")
  - `content` (text)
  - `context_source` (text)
  - `source_docs` (text — JSON string)
  - `created_at` (date)

### Rendering Markdown

The `response` field may contain Markdown (bold text, headers, bullet lists). To render it nicely in Bubble, consider using the **Rich Text Editor** element in read-only mode, or a plugin like **Markdown Element** or **Rich Text Display** to convert Markdown to formatted HTML.

### Loading State

The API typically responds in 3–8 seconds (embedding + vector search + Claude generation). Show a typing indicator or loading animation while waiting for the response.

### Error Handling

- If the API returns a non-200 status or the request times out, show a generic error message and offer a retry button.
- If `contextSource` is `"error"`, the `response` field will contain a user-friendly error message that you can display directly.

---

## Topic Scope

The assistant is designed to answer questions about:

- Estate planning and execution
- Wills, trusts, and probate
- Powers of attorney (medical and financial)
- State-specific probate thresholds and requirements
- Death notification procedures
- Asset management after death (financial, physical, real estate)
- Beneficiary management
- Funeral and memorial planning
- Creditor notice requirements
- Executor/personal representative duties
- The One Step After platform and roadmap features

Off-topic questions (weather, recipes, general trivia, etc.) are politely redirected by the assistant. No web search is performed for off-topic queries.

---

## Rate Limiting and Usage

Currently there are no rate limits enforced at the webhook level. For production, consider:

- Implementing rate limiting in Bubble (e.g., disable the send button for 3 seconds after each message)
- Tracking `usage.input_tokens` and `usage.output_tokens` from responses to monitor Claude API costs
- The Voyage AI embedding API has a rate limit of approximately 300 requests/minute on the free tier

---

## Testing

### Quick Test (curl)

```bash
curl -X POST "https://onestepafter.app.n8n.cloud/webhook/chat" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "What is an executor responsible for?",
    "sessionId": "test_001",
    "metadata": {
      "userName": "Test User",
      "userState": "CO"
    }
  }'
```

### Test Cases to Verify

| Test | Expected Behavior |
|------|-------------------|
| `"What are the probate thresholds in Colorado?"` | Returns answer from internal docs with source citations |
| `"What is a financial power of attorney?"` | Returns answer from POA Reference doc |
| `"What about in California?"` (with CO history) | Uses conversation context to understand follow-up |
| `"What is the best pizza in Denver?"` | Politely redirects to estate planning topics |
| `""` (empty message) | Returns 400 error |
| Very long message (2000+ chars) | Should still work, just slower |

---

## Architecture Summary

```
Bubble App
    │
    │  POST /webhook/chat
    │  { message, sessionId, conversationHistory, metadata }
    ▼
n8n Cloud (webhook)
    │
    ├─ Validate request
    ├─ Embed query → Voyage AI (voyage-3.5, 1024 dims)
    ├─ Vector search → Zilliz Cloud (cosine similarity, top 5)
    │
    ├─ If relevant docs found (score > 0.35):
    │   └─ Build context from matched document chunks
    │
    ├─ If no relevant docs AND on-topic:
    │   └─ Web search via Brave API (scoped to estate/probate)
    │
    ├─ If off-topic:
    │   └─ Skip search, provide redirect context
    │
    ├─ Send to Claude (claude-sonnet-4.5) with:
    │   - System prompt (One Step After scope + guidelines)
    │   - Document/web context
    │   - Conversation history
    │   - User's current message
    │
    └─ Return JSON response
         { response, contextSource, sessionId, sourceDocs, usage }
```

---

## Contact

For questions about the API behavior, system prompt changes, or adding new documents to the knowledge base, contact [your name/email here].
