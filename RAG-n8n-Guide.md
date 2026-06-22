# Building a RAG Customer Support Bot in n8n

A step-by-step guide to building **Retrieval-Augmented Generation (RAG)** pipelines in n8n using the Bitext Customer Support dataset. This guide covers two query approaches:

| Approach | Retrieval | Best for |
|----------|-----------|----------|
| **Vanilla RAG with Semantic Retrieval** | Pinecone vector search only | Simpler setup, LangChain AI Agent |
| **Hybrid RAG (BM25 + Semantic)** | Pinecone + Supabase BM25, fused with RRF + Cohere rerank | Keyword-heavy queries, exact term matches |

Both share the same **ingestion** workflow (CSV → Pinecone). Hybrid adds a **Supabase full-text search** index for BM25-style retrieval.

> **Viewing this guide:** Screenshots live in `screenshots/` and `screenshots/hybrid/`. On GitHub, use raw URLs such as `https://raw.githubusercontent.com/AliHassanSandhu/Rag-n8n/main/screenshots/hybrid/rag-hybrid-pipeline.png`.

---

## Table of Contents

1. [What is n8n?](#what-is-n8n)
2. [What is RAG?](#what-is-rag)
3. [Prerequisites](#prerequisites)
4. [Download n8n Workflow](#download-n8n-workflow)
5. [Dataset Overview](#dataset-overview)
6. [Architecture Overview](#architecture-overview)
7. [Workflow 1 — Upload and Process CSV](#workflow-1--upload-and-process-csv)
8. [Workflow 2 — Ingestion Pipeline (Upsert to Pinecone)](#workflow-2--ingestion-pipeline-upsert-to-pinecone)
9. [Workflow 3 — Vanilla RAG with Semantic Retrieval](#workflow-3--vanilla-rag-with-semantic-retrieval)
10. [Workflow 4 — Hybrid RAG (BM25 + Semantic)](#workflow-4--hybrid-rag-bm25--semantic)
11. [Expose Your Workflow with ngrok](#expose-your-workflow-with-ngrok)
12. [Testing](#testing)
13. [Troubleshooting](#troubleshooting)

Workflow 3 (vanilla) uses an AI Agent → parse JSON → SMTP. Workflow 4 (hybrid) uses parallel retrieval → RRF → Cohere → LLM Chain → SMTP.

---

## What is n8n?

**n8n** is an open-source workflow automation tool. You connect **nodes** on a visual canvas — each node performs one action (read a file, call an API, run code, etc.) — and data flows from one node to the next.

For AI/RAG use cases, n8n provides:

- **LangChain integration** — AI Agent, Embeddings, Vector Store, and Document Loader nodes
- **Triggers** — Webhooks, Forms, Chat, and schedules to start workflows
- **Self-hosting** — run locally on your machine with full control over files and API keys

Think of n8n as a visual pipeline builder: instead of writing a full Python script, you wire together triggers, data transforms, embedding models, and vector databases on a canvas.

---

## What is RAG?

**Retrieval-Augmented Generation (RAG)** combines:

1. **Retrieval** — search a knowledge base for relevant documents
2. **Augmentation** — inject those documents into the LLM prompt as context
3. **Generation** — the LLM produces an answer grounded in that context

For customer support, RAG lets a chatbot answer from your real Q&A data instead of inventing policies or phone numbers.

```
User question → Embed query → Search Pinecone → Retrieve top matches → LLM answers using retrieved responses
```

This guide uses **one ingestion workflow** and **two query options**:

| Workflow | Purpose |
|----------|---------|
| **Ingestion (upsert)** | Load CSV → embed → store in Pinecone (and Supabase for hybrid) |
| **Vanilla query** | Webhook → AI Agent → Pinecone semantic search → Groq → email |
| **Hybrid query** | Webhook → parallel Pinecone + Supabase BM25 → RRF → Cohere rerank → Groq → email |

---

## Prerequisites

### Software

- **n8n** installed locally (`n8n start` or Docker)
- **Node.js** (if running n8n via npm)
- **ngrok** (optional — to expose local webhooks to the internet)
- **SMTP mail account** (e.g. Gmail with App Password) — to email support responses to the user

### Dataset

- `Bitext_Sample_Customer_Support_Training_Dataset_27K_responses-v11.csv`
- ~26,872 rows of customer support Q&A pairs

### API Keys

Create accounts and obtain the following keys. Store them in n8n **Credentials** (never commit keys to git).

| Key | Service | Used For |
|-----|---------|----------|
| `pinecone_api_key` | [Pinecone](https://app.pinecone.io/) | Vector database — store and search embeddings |
| `groq_api_key` | [Groq](https://console.groq.com/) | Chat model — generate answers (`openai/gpt-oss-120b`) |
| `huggingface_api_key` | [Hugging Face](https://huggingface.co/settings/tokens) | Embedding model — convert text to vectors |
| `cohere_api_key` | [Cohere](https://dashboard.cohere.com/) | Reranker — reorder Pinecone results for better relevance |
| `ngrok_api_key` | [ngrok](https://ngrok.com/) | Tunnel — expose local n8n webhooks publicly |
| Supabase service role key | [Supabase](https://supabase.com/) | BM25 full-text search (hybrid workflow only) |

### Pinecone Index

Create an index in the Pinecone console **before** ingesting data:

| Setting | Value |
|---------|--------|
| **Name** | e.g. `bitext-support` |
| **Dimensions** | **768** (matches Hugging Face `sentence-transformers/all-mpnet-base-v2`) |
| **Metric** | `cosine` |
| **Type** | Serverless |

> **Note:** `nomic-ai/nomic-embed-text-v1.5` is **not** available on Hugging Face's free Inference API ("No Inference Provider available"). Use `sentence-transformers/all-mpnet-base-v2` instead (768 dimensions). Groq's `openai/gpt-oss-120b` is a **chat model only** — it cannot be used for embeddings.

---

## Download n8n Workflow

Ready-made n8n workflow exports are available for import:

| Workflow | File | Description |
|----------|------|-------------|
| **Vanilla RAG** | `RAG Workflow.json` | Ingestion + semantic-only AI Agent query pipeline |
| **Hybrid RAG** | Export from your n8n canvas (Workflow 4) | BM25 + semantic parallel retrieval |

**[Download RAG Workflow.json — Vanilla (Google Drive)](https://drive.google.com/file/d/18zXZqNQU4VsijJ3nZYlMTBY8gZXecLN5/view?usp=sharing)**

### How to import

1. Open n8n → **Workflows** → **Add workflow** → **Import from File** (or use the three-dot menu → **Import from File**).
2. Select the downloaded `RAG Workflow.json`.
3. Open each node and connect your credentials:
   - Pinecone (`pinecone_api_key`)
   - Groq (`groq_api_key`)
   - Hugging Face (`huggingface_api_key`)
   - Cohere (`cohere_api_key`)
4. Set your Pinecone index name and namespace to match your setup.
5. Activate the workflow and test with a small CSV upload first.

---

## Dataset Overview

The Bitext CSV has 5 columns:

| Column | Description | Example |
|--------|-------------|---------|
| `instruction` | Customer question | `I need to cancel purchase {{Order Number}}` |
| `response` | Support agent answer | Step-by-step cancellation instructions |
| `category` | Topic | `ORDER`, `REFUND`, `ACCOUNT`, … |
| `intent` | Specific intent | `cancel_order`, `track_refund`, … |
| `flags` | Query style metadata | `BL`, `BLQZ` (typos, casing — optional for RAG) |

Each row is a self-contained Q&A pair — no extra chunking is required.

Responses contain placeholder tokens like `{{Order Number}}` and `{{Website URL}}`. Replace these with real company values in the Code node if needed.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                  INGESTION WORKFLOW (one-time)               │
│                                                              │
│  Form Trigger → Extract CSV → Code → Pinecone (Insert)      │
│                                         ├── HF Embeddings    │
│                                         └── Default Data     │
│                                              Loader          │
│  (Hybrid only) Code → Supabase insert into support_docs     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│     VANILLA QUERY — Semantic Retrieval (Workflow 3)          │
│                                                              │
│  Google Form → Apps Script → ngrok → Webhook → AI Agent       │
│               ├── Groq Chat Model                            │
│               └── Pinecone (Retrieve Tool)                   │
│                        ├── HF Embeddings (same model)        │
│                        └── Cohere Reranker                   │
│  AI Agent → Code (parse JSON) → SMTP (email response)        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│     HYBRID QUERY — BM25 + Semantic (Workflow 4)              │
│                                                              │
│  Webhook → Edit Fields (query, email)                        │
│       ├── Branch A: HF Embedding → Pinecone /query           │
│       └── Branch B: Supabase RPC search_support_docs         │
│              → Merge → RRF Fusion → Cohere Rerank            │
│              → Combine Context → LLM Chain (Groq) → SMTP     │
└─────────────────────────────────────────────────────────────┘
```

---

## Workflow 1 — Upload and Process CSV

This workflow reads the CSV uploaded via an n8n Form and transforms each row into the document format Pinecone expects.

![Upload and Process CSV workflow](https://github.com/AliHassanSandhu/Rag-n8n/blob/main/screenshots/upload-and-process-csv.png)

### Step 1: Form Trigger — Upload CSV

1. Add a **Form Trigger** node (or **On form submission**).
2. Add a **File** field, e.g. `upload_csv_file`.
3. Publish the form URL and upload your CSV.

### Step 2: Extract from File

Add **Extract from File**:

| Setting | Value |
|---------|--------|
| **Operation** | Extract From CSV |
| **Input Binary Field** | `upload_csv_file` (must match the form field name — **not** the filename) |
| **Header Row** | true |
| **Delimiter** | `,` |

Each row becomes one item with JSON fields: `flags`, `instruction`, `category`, `intent`, `response`.

### Step 3: Code Node — Build Documents

Add a **Code** node. Set mode to **Run once for all items**.

```javascript
return $input.all().map((item) => {
  const row = item.json;

  const question = row.instruction ?? "";
  const answer = row.response ?? "";

  return {
    json: {
      // Text that gets embedded (searchable)
      pageContent: `Category: ${row.category}\nIntent: ${row.intent}\nQuestion: ${question}`,
      metadata: {
        category: row.category,
        intent: row.intent,
        instruction: question,
        response: answer,
        flags: row.flags,
      },
    },
  };
});
```

**Design choice:**

- `pageContent` = what gets **embedded and searched** (category + intent + question)
- `metadata.response` = the support answer retrieved at query time for the LLM

> For initial testing, add a **Limit** node (50–500 items) between Extract from File and Code before running the full 27K rows.

---

## Workflow 2 — Ingestion Pipeline (Upsert to Pinecone)

Connect the Code node output to **Pinecone Vector Store** in **Insert Documents** mode, with Embeddings and Default Data Loader sub-nodes.

![Ingestion Pipeline workflow](https://github.com/AliHassanSandhu/Rag-n8n/blob/main/screenshots/ingestion-pipeline.png)

### Step 4: Embeddings HuggingFace Inference

Add **Embeddings HuggingFace Inference** and connect it to the **Embedding** port on Pinecone.

| Setting | Value |
|---------|--------|
| **Credential** | Hugging Face API token (`huggingface_api_key`) |
| **Model** | `sentence-transformers/all-mpnet-base-v2` |

Token must have **"Make calls to Inference Providers"** permission.

### Step 5: Default Data Loader

Add **Default Data Loader** and connect it to the **Document** port on Pinecone.

| Setting | Value |
|---------|--------|
| **Mode** | **Load Specific Data** |
| **Text** | `={{ $json.pageContent }}` |
| **Text Splitting** | **Custom** → connect **Character Text Splitter** with **Chunk Size: 10000** |

Do **not** use **Load All Input Data** — that causes blob/line parsing and stores wrong metadata (`blobType`, `line`, `source: blob`).

#### Metadata — add one property per field

The Metadata section uses **Name / Value** pairs. Do **not** put the whole object in the Name field.

| Name | Value |
|------|--------|
| `category` | `={{ $json.metadata.category }}` |
| `intent` | `={{ $json.metadata.intent }}` |
| `instruction` | `={{ $json.metadata.instruction }}` |
| `response` | `={{ $json.metadata.response }}` |
| `flags` | `={{ $json.metadata.flags }}` |

Click **Add property** for each row. Name is a plain string; Value is an expression.

### Step 6: Pinecone Vector Store — Insert Documents

| Setting | Value |
|---------|--------|
| **Credential** | Pinecone API key (`pinecone_api_key`) |
| **Operation Mode** | Insert Documents |
| **Pinecone Index** | your index (e.g. `bitext-support`) |
| **Embedding Batch Size** | `200` |
| **Namespace** | e.g. `bitext-v1` (optional) |
| **Clear Namespace** | `true` on first run only |

### Step 7: Split in Batches (full dataset)

For all 26,872 rows, add **Split in Batches** (batch size 100–200) between Code and Pinecone, and loop until complete.

```
Form → Extract CSV → Code → Split in Batches → Pinecone (Insert)
                                    ↑___________________|
```

### Verify Insert

In the Pinecone console, open one vector. Metadata should look like:

```json
{
  "category": "ORDER",
  "intent": "cancel_order",
  "instruction": "I need to cancel purchase {{Order Number}}",
  "response": "To cancel your purchase, please follow these steps...",
  "flags": "BL"
}
```

If you see `blobType`, `line`, or `source: blob`, the Default Data Loader is misconfigured — fix metadata and re-index.

### Hybrid only — Supabase BM25 index

For **Workflow 4 (Hybrid RAG)**, also upsert rows into Supabase so keyword search can run in parallel with Pinecone.

1. In Supabase SQL Editor, create the table and search function:

```sql
create table if not exists support_docs (
  doc_id text primary key,
  category text,
  intent text,
  instruction text,
  response text,
  search_text text,
  fts tsvector generated always as (to_tsvector('english', coalesce(search_text, ''))) stored
);

create index if not exists support_docs_fts_idx on support_docs using gin (fts);

create or replace function search_support_docs(query_text text, match_count int default 10)
returns table (
  doc_id text,
  category text,
  intent text,
  instruction text,
  response text,
  search_text text,
  rank real
)
language sql stable
as $$
  select
    doc_id, category, intent, instruction, response, search_text,
    ts_rank(fts, websearch_to_tsquery('english', query_text)) as rank
  from support_docs
  where fts @@ websearch_to_tsquery('english', query_text)
  order by rank desc
  limit match_count;
$$;
```

2. After the Code node in Workflow 1, add a **Supabase** node (or extend the Code node to output both Pinecone and Supabase fields):

| Supabase field | Source |
|----------------|--------|
| `doc_id` | `={{ $json.metadata.category }}_{{ $json.metadata.intent }}_{{ $runIndex }}` |
| `search_text` | Same as `pageContent` (category + intent + question) |
| `response`, `category`, `intent`, `instruction` | From metadata |

3. Supabase credential **Host** must be `https://YOUR-PROJECT.supabase.co` — do **not** append `/rest/v1/`.

---

## Workflow 3 — Vanilla RAG with Semantic Retrieval

This workflow answers user questions using **semantic search only** — Pinecone vector similarity via the LangChain **AI Agent** retrieve-as-tool pattern, then Groq for generation.

![Vanilla RAG — Google Form, AI Agent, Pinecone semantic retrieval, and SMTP email](https://raw.githubusercontent.com/AliHassanSandhu/Rag-n8n/main/screenshots/rag.png)

### Step 8: Webhook Trigger (Google Forms)

This workflow is triggered by a **Google Form** submission. A Google Apps Script sends the form answers as JSON to your n8n webhook via **ngrok** (so Google can reach your local n8n instance).

#### 8.1 — Configure the n8n Webhook node

1. Add a **Webhook** node as the first node in Workflow 3.
2. Set **HTTP Method** to `POST`.
3. Set **Authentication** to `None` (Google Apps Script sends plain JSON).
4. Set **Respond** to `When Last Node Finishes` (or `Immediately` if you only need to log the trigger).
5. Copy the webhook URLs from the node panel:

| URL type | Path | When it works |
|----------|------|----------------|
| **Test URL** | `/webhook-test/<id>` | Only while **Listen for test event** is active in n8n |
| **Production URL** | `/webhook/<id>` | When the workflow is **Active** (published) |

Example test URL (replace with yours from the Webhook node):

```
https://YOUR-SUBDOMAIN.ngrok-free.dev/webhook-test/03993e47-689e-4970-a743-3a825ca32291
```

Example production URL:

```
https://YOUR-SUBDOMAIN.ngrok-free.dev/webhook/03993e47-689e-4970-a743-3a825ca32291
```

> **Important:** Google Forms runs in the cloud and cannot call `localhost`. You must expose n8n with **ngrok** (see [Expose Your Workflow with ngrok](#expose-your-workflow-with-ngrok)) and use the **HTTPS** ngrok URL in Apps Script.

#### 8.2 — Create the Google Form

1. Go to [Google Forms](https://forms.google.com/) and create a new form.
2. Add a **Short answer** question titled **`Email`** (for the reply address).
3. Add a **Short answer** question for the support query, e.g. title: **`Question`** or **`Query`**.
4. (Optional) Add more fields (name, category).
5. Click **Send** → link icon → copy the form URL for testers.

Remember the **exact question titles** — Apps Script uses them as JSON keys.

#### 8.3 — Add Google Apps Script

1. In the Form editor, open **⋮** → **Script editor** (or go to [script.google.com](https://script.google.com) and bind the script to the form).
2. Paste this script and replace `webhookUrl` with your ngrok + n8n webhook URL:

```javascript
function onFormSubmit(e) {
  // Use webhook-test URL while testing (Listen for test event ON in n8n)
  // Use /webhook/ URL when the n8n workflow is Active (production)
  const webhookUrl =
    "https://YOUR-SUBDOMAIN.ngrok-free.dev/webhook-test/03993e47-689e-4970-a743-3a825ca32291";

  const responses = e.response.getItemResponses();
  const payload = {};

  responses.forEach(item => {
    payload[item.getItem().getTitle()] = item.getResponse();
  });

  Logger.log(JSON.stringify(payload));

  const response = UrlFetchApp.fetch(webhookUrl, {
    method: "post",
    contentType: "application/json",
    payload: JSON.stringify(payload),
    muteHttpExceptions: true,
    headers: {
      "ngrok-skip-browser-warning": "true",
    },
  });

  Logger.log("Status: " + response.getResponseCode());
  Logger.log(response.getContentText());
}
```

3. **Save** the project (e.g. name: `Send to n8n Webhook`).
4. Run **Triggers** (clock icon) → **Add Trigger**:
   - Function: `onFormSubmit`
   - Event source: **From form**
   - Event type: **On form submit**
5. Authorize the script when prompted (Google needs permission to call external URLs).

Example payload sent to n8n (keys match your form question titles):

```json
{
  "Email": "user@example.com",
  "Question": "How do I cancel my order?"
}
```

#### 8.4 — Map form data to the AI Agent

In the **AI Agent** node, set the user message from the webhook body. If your form question title is **`Question`**:

```
={{ $json.body.Question }}
```

If n8n puts fields at the root (depends on Webhook version/settings), try:

```
={{ $json.Question }}
```

Use the **Webhook node output** after a test submission to confirm the exact path.

#### 8.5 — Start ngrok and n8n

1. Start n8n locally (default port `5678`):

```powershell
n8n start
```

2. In a second terminal, start ngrok:

```bash
ngrok http 5678
```

3. Copy the **Forwarding** HTTPS URL (e.g. `https://borrowing-suds-unharmed.ngrok-free.dev`).
4. Build the full webhook URL: `https://<ngrok-host>/webhook-test/<your-webhook-id>` for testing.

> Free ngrok URLs change each time you restart ngrok unless you use a reserved domain. Update `webhookUrl` in Apps Script whenever the ngrok URL changes.

#### 8.6 — Test the webhook end-to-end

**Test A: n8n only (no Google Form)**

1. Open Workflow 3 in n8n.
2. Click **Listen for test event** on the Webhook node.
3. Send a test POST with curl or Postman:

```bash
curl -X POST "https://YOUR-SUBDOMAIN.ngrok-free.dev/webhook-test/03993e47-689e-4970-a743-3a825ca32291" \
  -H "Content-Type: application/json" \
  -H "ngrok-skip-browser-warning: true" \
  -d "{\"Question\": \"How do I cancel my order?\"}"
```

4. Confirm the Webhook node receives the payload and the AI Agent runs.
5. Check **Executions** for retrieved Pinecone documents and the final answer.

**Test B: Google Form submission**

1. In n8n: **Listen for test event** on the Webhook node (for test URL) **or** **Activate** the workflow (for production `/webhook/` URL).
2. Ensure ngrok is running and Apps Script `webhookUrl` matches the current ngrok + webhook path.
3. Open the Google Form and submit: *How do I cancel my order?*
4. In n8n → **Executions**, verify a new run appeared.
5. In Apps Script → **Executions** / **Logs**, confirm status `200` and no errors.

**Test C: Switch to production**

1. **Activate** Workflow 3 in n8n.
2. Change Apps Script `webhookUrl` from `/webhook-test/` to `/webhook/`:

```javascript
const webhookUrl =
  "https://YOUR-SUBDOMAIN.ngrok-free.dev/webhook/03993e47-689e-4970-a743-3a825ca32291";
```

3. Submit the form again — it should trigger without **Listen for test event**.

**Troubleshooting webhook tests**

| Issue | Fix |
|-------|-----|
| Form submits but n8n shows nothing | ngrok not running, wrong URL, or using test URL without **Listen for test event** |
| Apps Script error / 404 | URL path typo; use `/webhook-test/` vs `/webhook/` correctly |
| ngrok browser warning page | Add header `ngrok-skip-browser-warning: true` in Apps Script (included in script above) |
| AI Agent gets empty input | Wrong expression — inspect Webhook output and match form field title |
| 401 / 403 from n8n | Remove Webhook authentication or match credentials in Apps Script |

### Step 9: AI Agent

Add **AI Agent** and connect the Webhook.

**System message example:**

Configure the AI Agent to return a **JSON object** (as a string in the `output` field) so the next Code node can parse it and send an email.

```
You are a customer support assistant.

When a user asks a question:
1. Always search the knowledge base tool first.
2. Answer ONLY using the retrieved support responses (metadata.response).
3. Be concise and friendly.
4. If nothing relevant is found, say you don't know and offer human support.
5. Do not invent policies, phone numbers, or URLs.

Return your final answer ONLY as valid JSON with these exact keys:
{
  "Email": "<user email from the form>",
  "Query": "<user question>",
  "Output": "<your support answer>"
}
```

Ensure the Google Form includes an **Email** field — Apps Script sends it in the webhook payload alongside **Question** (or your query field title).

### Step 10: Groq Chat Model (required)

Connect **Groq Chat Model** to the **Chat Model** port.

| Setting | Value |
|---------|--------|
| **Credential** | Groq API key (`groq_api_key`) |
| **Model** | `openai/gpt-oss-120b` or `llama-3.3-70b-versatile` |

Groq is used **only for generating answers**, not for embeddings.

### Step 11: Pinecone Vector Store — Retrieve Tool

Connect **Pinecone Vector Store** to the **Tool** port.

| Setting | Value |
|---------|--------|
| **Operation Mode** | **Retrieve Documents (As Tool for AI Agent)** |
| **Pinecone Index** | Same index as insert |
| **Namespace** | Same as insert (e.g. `bitext-v1`) |
| **Limit / Top K** | `5` |

**Tool description:**

```
Search the customer support knowledge base for answers about orders, refunds, shipping, accounts, and payments. Always use this tool before answering.
```

Connect the **same Embeddings HuggingFace** model used during insert (`sentence-transformers/all-mpnet-base-v2`).

### Step 12: Cohere Reranker (optional but recommended)

Connect **Cohere Reranker** to the **Reranker** port on Pinecone.

| Setting | Value |
|---------|--------|
| **Credential** | Cohere API key (`cohere_api_key`) |
| **Top N** | `3` |

Reranking reorders Pinecone's initial results so the AI Agent receives the most relevant documents first.

### Step 13: Memory (optional)

For multi-turn conversations, connect **Window Buffer Memory** to the Memory port. Skip for single-question webhooks.

### Step 14: AI Agent output format

After the AI Agent runs, n8n returns output in this structure:

```json
[
  {
    "output": "{
  "Email": "asd@gmail.com",
  "Query": "How to cancel order",
  "Output": "To cancel an order: 1. Log into your account. 2. Navigate to the 'Order Details' page. 3. Click the 'Cancel Order' button if available for your order. 4. If cancellation is not possible, contact our support team at support@example.com with your order number for assistance."
}"
  }
]
```

The `output` field is a **JSON string** (not a parsed object). The next Code node parses it into separate fields for the email node.

> **Tip:** In the AI Agent system prompt, require JSON with keys `Email`, `Query`, and `Output` so parsing is reliable. Match key names to what your Google Form sends (e.g. form field title `Email` → `"Email"` in JSON).

### Step 15: Code node — parse AI output

Add a **Code** node after the AI Agent. Set mode to **Run once for each item**.

```javascript
const data = JSON.parse($json.output);

return [{
  json: {
    email: data.Email,
    query: data.Query,
    answer: data.Output
  }
}];
```

| Input field | Output field | Used by |
|-------------|--------------|---------|
| `output` (JSON string) | `email`, `query`, `answer` | SMTP node |

If parsing fails, check the AI Agent execution log — the model may have returned markdown or extra text around the JSON. Tighten the system prompt to **JSON only**.

### Step 16: SMTP — email the support response

Add an **Send Email (SMTP)** node after the Code node to deliver the answer to the user.

![SMTP node configuration for customer support email response](https://raw.githubusercontent.com/AliHassanSandhu/Rag-n8n/main/screenshots/smtp-email-node.png)

| Setting | Value |
|---------|--------|
| **Credential** | SMTP account (Gmail, Outlook, or your mail provider) |
| **To Email** | `={{ $json.email }}` |
| **Subject** | `Support Response: {{ $json.query }}` |
| **Email Format** | HTML |
| **HTML** | See template below |

**HTML body template** (paste into the SMTP node **HTML** field):

```html
<html>
<body style="font-family: Arial, sans-serif; line-height:1.6;">
    <h2>Customer Support Response</h2>

    <p><strong>Query:</strong></p>
    <p>{{ $json.query }}</p>

    <hr>

    <p><strong>Response:</strong></p>
    <div>
        {{ $json.answer.replace(/\n/g, '<br>') }}
    </div>

    <br>
    <hr>

    <p>
        Regards,<br>
        Customer Support Team
    </p>
</body>
</html>
```

The `.replace(/\n/g, '<br>')` expression converts line breaks in the AI answer into HTML `<br>` tags so multi-step responses render correctly in the email.

#### SMTP credential setup (Gmail example)

1. In n8n → **Credentials** → **SMTP** → create new.
2. **User:** your Gmail address.
3. **Password:** a [Google App Password](https://myaccount.google.com/apppasswords) (not your regular Gmail password).
4. **Host:** `smtp.gmail.com` | **Port:** `465` | **SSL/TLS:** ON.

Test the SMTP node alone with hardcoded `To Email` before connecting the full pipeline.

#### Example output — email received by the user

After a successful run (Google Form → AI Agent → Code → SMTP), the user receives an HTML email like this:

![Example customer support email response sent automatically via n8n SMTP](https://raw.githubusercontent.com/AliHassanSandhu/Rag-n8n/main/screenshots/email-response.jpeg)

The email shows the original **Query**, the AI-generated **Response** (grounded in Pinecone retrieval), and the **Customer Support Team** sign-off. The footer `This email was sent automatically with n8n` is added by n8n when using the SMTP node.

### Canvas Summary (Vanilla)

```
Google Form → Apps Script → ngrok → Webhook → AI Agent
                                                  ├── Chat Model      → Groq Chat Model
                                                  ├── Tool            → Pinecone (Retrieve as Tool)
                                                  │                         ├── Embeddings HF
                                                  │                         └── Cohere Reranker
                                                  └── Memory            → Window Buffer Memory (optional)
                                        ↓
                                   Code (parse JSON output)
                                        ↓
                                   SMTP (email response)
```

---

## Workflow 4 — Hybrid RAG (BM25 + Semantic)

This workflow improves retrieval by running **two searches in parallel** and fusing the results:

- **Branch A (Semantic)** — embed the query with Hugging Face, query Pinecone by vector similarity
- **Branch B (BM25)** — call Supabase `search_support_docs()` for PostgreSQL full-text search

Results are merged with **Reciprocal Rank Fusion (RRF)**, reranked with **Cohere**, collapsed into one context block, and passed to a **Basic LLM Chain** (Groq) instead of an AI Agent.

![Hybrid RAG pipeline — parallel Pinecone + Supabase, RRF, Cohere rerank, Groq LLM Chain, SMTP](https://raw.githubusercontent.com/AliHassanSandhu/Rag-n8n/main/screenshots/hybrid/rag-hybrid-pipeline.png)

### Prerequisites (hybrid)

- Pinecone index populated (Workflow 2)
- Supabase `support_docs` table + `search_support_docs` function (see [Hybrid only — Supabase BM25 index](#hybrid-only--supabase-bm25-index))
- Same API keys as vanilla, plus Supabase service role key

### Step 17: Webhook + Edit Fields

Reuse the same **Google Form → Apps Script → ngrok → Webhook** setup from [Step 8](#step-8-webhook-trigger-google-forms). After the Webhook, add an **Edit Fields** (Set) node to normalize `query` and `email` for downstream branches.

![Hybrid — Webhook trigger and Edit Fields node](https://raw.githubusercontent.com/AliHassanSandhu/Rag-n8n/main/screenshots/hybrid/webhook-edit-fields.png)

| Field | Expression (adjust to match your form titles) |
|-------|-----------------------------------------------|
| `query` | `={{ $json.body.Question }}` |
| `email` | `={{ $json.body.Email }}` |

Inspect the Webhook output after a test submission — use `$json.Question` instead of `$json.body.Question` if fields are at the root.

### Step 18: Branch A — Semantic (Pinecone)

From **Edit Fields**, connect **Branch A**: embed the query, then query Pinecone's REST API directly (HTTP Request nodes give full control over the 768-dim vector).

![Hybrid Branch A — Hugging Face embedding and Pinecone vector query](https://raw.githubusercontent.com/AliHassanSandhu/Rag-n8n/main/screenshots/hybrid/branch-a-semantic.png)

#### 18.1 — Hugging Face Embedding (HTTP Request)

| Setting | Value |
|---------|--------|
| **Method** | POST |
| **URL** | `https://router.huggingface.co/hf-inference/models/sentence-transformers/all-mpnet-base-v2/pipeline/feature-extraction` |
| **Authentication** | Header Auth — `Authorization: Bearer YOUR_HF_TOKEN` |
| **Send Body** | ON |
| **Body Content Type** | JSON |
| **JSON body** | `={{ JSON.stringify({ inputs: $('Edit Fields').first().json.query }) }}` |

> Do **not** use the deprecated `api-inference.huggingface.co` URL — it returns errors. **Send Body** must be ON or the request sends invalid JSON.

#### 18.2 — Build Pinecone query body (Code)

Add a **Code** node to extract the 768-dimensional embedding array from the HF response and build the Pinecone `/query` payload:

```javascript
const embedding = $input.first().json;
const vector = Array.isArray(embedding[0]) ? embedding[0] : embedding;

return [{
  json: {
    vector,
    topK: 10,
    includeMetadata: true,
    namespace: '__default__'
  }
}];
```

#### 18.3 — Pinecone query (HTTP Request)

| Setting | Value |
|---------|--------|
| **Method** | POST |
| **URL** | `https://YOUR-INDEX-HOST/query` |
| **Authentication** | Header — `Api-Key: YOUR_PINECONE_KEY` |
| **Send Body** | ON |
| **Body** | `={{ JSON.stringify({ vector: $json.vector, topK: $json.topK, includeMetadata: true, namespace: $json.namespace }) }}` |

Pinecone returns `{ matches: [ { id, score, metadata }, ... ] }`.

### Step 19: Branch B — BM25 (Supabase)

From **Edit Fields**, connect **Branch B**: call the Supabase RPC function for full-text search.

![Hybrid Branch B — Supabase RPC search_support_docs](https://raw.githubusercontent.com/AliHassanSandhu/Rag-n8n/main/screenshots/hybrid/branch-b-bm25.png)

Add a **Supabase** node:

| Setting | Value |
|---------|--------|
| **Resource** | Database |
| **Operation** | Execute RPC |
| **Function** | `search_support_docs` |
| **Send Body** | **ON** |
| **Parameters** | `query_text`: `={{ $('Edit Fields').first().json.query }}`, `match_count`: `10` |

Common errors:

| Error | Fix |
|-------|-----|
| `PGRST202` — function not found | Create `search_support_docs` in Supabase SQL Editor (see ingestion section) |
| 404 without parameters | Turn **Send Body** ON and pass `query_text` + `match_count` |

Supabase returns one item per matched row (`doc_id`, `response`, `rank`, etc.).

### Step 20: Merge, RRF, Cohere rerank, Combine Context

Connect both branches into a **Merge** node (mode: **Append**), then fuse and rerank.

![Hybrid — Merge, RRF fusion, Cohere rerank, Combine Context](https://raw.githubusercontent.com/AliHassanSandhu/Rag-n8n/main/screenshots/hybrid/merge-rrf-cohere-combine.png)

#### 20.1 — RRF Fusion (Code)

Add a **Code** node (e.g. named `Code in JavaScript1`) to combine Pinecone matches and Supabase rows with Reciprocal Rank Fusion:

```javascript
const K = 60;
const scores = new Map();

function normalizeId(id) {
  if (!id) return '';
  return String(id).replace(/^[\t\n=]+/, '').trim();
}

function addResult(id, rank, text, metadata) {
  const key = normalizeId(id);
  if (!key) return;
  const rrf = 1 / (K + rank);
  const existing = scores.get(key) || { score: 0, text: '', metadata: {} };
  existing.score += rrf;
  if (text) existing.text = text;
  if (metadata) existing.metadata = { ...existing.metadata, ...metadata };
  scores.set(key, existing);
}

// Pinecone branch: one item with matches[]
const pineconeItem = $('Pinecone Query').first().json; // rename to your HTTP node name
(pineconeItem.matches || []).forEach((m, i) => {
  addResult(
    m.metadata?.id || m.id,
    i + 1,
    m.metadata?.response || '',
    m.metadata || {}
  );
});

// Supabase branch: multiple flat items
$('Supabase').all().forEach((item, i) => {
  const row = item.json;
  addResult(row.doc_id, i + 1, row.response, row);
});

const fused = [...scores.entries()]
  .sort((a, b) => b[1].score - a[1].score)
  .slice(0, 10)
  .map(([id, data]) => ({
    json: {
      id,
      rrf_score: data.score,
      text: data.text,
      metadata: data.metadata
    }
  }));

return fused;
```

> Adjust `$('Pinecone Query')` and `$('Supabase')` to match your actual node names. Normalize IDs to handle legacy data with leading `=` or tab/newline characters.

#### 20.2 — Cohere Rerank (HTTP Request)

| Setting | Value |
|---------|--------|
| **Method** | POST |
| **URL** | `https://api.cohere.com/v1/rerank` |
| **Headers** | `Authorization: Bearer YOUR_COHERE_KEY` |
| **Send Body** | ON |
| **JSON body** | See expression below |

Build the documents array from the RRF node — use `$('Code in JavaScript1').all()`, not `$input`:

```javascript
={{
  JSON.stringify({
    model: 'rerank-english-v3.0',
    query: $('Edit Fields').first().json.query,
    documents: $('Code in JavaScript1').all().map(i => i.json.text),
    top_n: 3
  })
}}
```

Avoid `==` in n8n expressions (use single `=`).

#### 20.3 — Map reranked results (Code)

Map Cohere's `results` array back to your document metadata:

```javascript
const reranked = $input.first().json.results || [];
const sources = $('Code in JavaScript1').all();

return reranked.map(r => ({
  json: {
    ...sources[r.index].json,
    cohere_score: r.relevance_score
  }
}));
```

#### 20.4 — Combine Context (Code)

The LLM Chain runs **once per input item**. Collapse the top reranked docs into **one item** so Groq is called only once:

```javascript
const docs = $input.all();
const query = $('Edit Fields').first().json.query;
const email = $('Edit Fields').first().json.email;

const context = docs
  .map((d, i) => `[${i + 1}] ${d.json.text}`)
  .join('\n\n');

return [{
  json: {
    query,
    email,
    context,
    doc_count: docs.length
  }
}];
```

### Step 21: LLM Chain (Groq)

Add a **Basic LLM Chain** connected to **Groq Chat Model**.

![Hybrid — Basic LLM Chain with Groq chat model](https://raw.githubusercontent.com/AliHassanSandhu/Rag-n8n/main/screenshots/hybrid/llm-chain-grok.png)

| Setting | Value |
|---------|--------|
| **Prompt** | See template below |
| **Chat Model** | Groq — `openai/gpt-oss-120b` or `llama-3.3-70b-versatile` |

**Prompt template:**

```
You are a customer support assistant.

Answer the user's question using ONLY the support responses below.
If the context does not contain the answer, say you don't know and offer human support.
Do not invent policies, phone numbers, or URLs.

User question: {{ $json.query }}

Support context:
{{ $json.context }}

Return your answer as JSON with exactly these keys:
{"Email":"<user email>","Query":"<user question>","Output":"<your answer>"}
```

Connect **Combine Context** directly to the LLM Chain — not the three separate reranked items.

### Step 22: Extract answer + SMTP email

Add a **Code** node to parse the LLM output, then **Send Email (SMTP)** — same pattern as vanilla Workflow 3.

![Hybrid — Extract Answer Code node and SMTP email](https://raw.githubusercontent.com/AliHassanSandhu/Rag-n8n/main/screenshots/hybrid/format-send-email.png)

**Extract Answer (Code):**

```javascript
const raw = $input.first().json.text || $input.first().json.output || '';
const email = $('Combine Context').first().json.email;
const query = $('Combine Context').first().json.query;

// Pull JSON from LLM output (handles markdown fences)
const match = raw.match(/\{[\s\S]*"Output"[\s\S]*\}/);
let answer = raw;
if (match) {
  try {
    const parsed = JSON.parse(match[0]);
    answer = parsed.Output || raw;
  } catch (e) {
    answer = raw;
  }
}

return [{
  json: { email, query, answer }
}];
```

Reuse the SMTP HTML template from [Step 16](#step-16-smtp--email-the-support-response).

### Canvas Summary (Hybrid)

```
Webhook → Edit Fields (query, email)
    ├── Branch A: HF Embedding → Build Pinecone Body → Pinecone /query
    └── Branch B: Supabase RPC search_support_docs
              → Merge (Append)
              → RRF Fusion (Code)
              → Cohere Rerank (HTTP)
              → Map Reranked (Code)
              → Combine Context (Code, 1 item)
              → LLM Chain (Groq)
              → Extract Answer (Code)
              → SMTP (email)
```

### Vanilla vs Hybrid — when to use which

| | Vanilla RAG | Hybrid RAG |
|---|-------------|------------|
| **Retrieval** | Pinecone semantic only | Pinecone + Supabase BM25 |
| **Fusion** | Cohere rerank inside Pinecone tool | RRF then Cohere rerank |
| **Generation** | AI Agent (tool-calling) | Basic LLM Chain (fixed context) |
| **Complexity** | Lower — LangChain handles retrieve | Higher — manual HTTP + merge |
| **Best for** | Prototyping, natural-language questions | Order IDs, exact keywords, policy terms |

---

## Expose Your Workflow with ngrok

Local n8n webhooks are only reachable on your machine. **Google Forms** and Apps Script run on Google's servers, so they need a **public HTTPS URL** to POST to your webhook. **ngrok** creates a secure tunnel to your local n8n.

### Setup

1. Install [ngrok](https://ngrok.com/download) and authenticate:

```bash
ngrok config add-authtoken YOUR_NGROK_API_KEY
```

2. Start n8n (default port `5678`):

```powershell
n8n start
```

3. In another terminal, start the tunnel:

```bash
ngrok http 5678
```

4. Copy the **Forwarding** HTTPS URL from the ngrok console, e.g.:

```
https://borrowing-suds-unharmed.ngrok-free.dev
```

5. Combine with your n8n Webhook path from the Webhook node:

| Mode | Full URL pattern |
|------|------------------|
| Testing | `https://<ngrok-host>/webhook-test/<webhook-id>` |
| Production (workflow Active) | `https://<ngrok-host>/webhook/<webhook-id>` |

6. Paste that full URL into the Google Apps Script `webhookUrl` variable (see [Step 8 — Webhook Trigger (Google Forms)](#step-8-webhook-trigger-google-forms)).

### Notes

- Free ngrok URLs **change** when you restart ngrok — update Apps Script each time.
- Keep both **ngrok** and **n8n** running while testing Google Forms.
- For a stable URL, use an ngrok **reserved domain** (paid plan) or deploy n8n to a cloud host with a permanent URL.

---

## Testing

### Ingestion

1. Upload CSV via Form Trigger.
2. Run with **Limit 50** first.
3. Confirm ~50 vectors in Pinecone with correct metadata (including `response`).
4. Remove Limit, run full dataset with Split in Batches.

### Query — Vanilla RAG

**Option 1 — Google Form:** Submit the form with a question like *How do I cancel my order?* (see [Step 8.6 — Test the webhook end-to-end](#86--test-the-webhook-end-to-end)).

**Option 2 — curl / Postman:** Send a POST to your ngrok webhook URL with the same JSON shape Apps Script sends (keys = form question titles):

```json
{
  "Email": "user@example.com",
  "Question": "How do I cancel my order?"
}
```

**Option 3 — Verify email:** After the run completes, check the SMTP node execution and the recipient inbox for the HTML email with query and answer.

### Query — Hybrid RAG

1. Confirm Supabase has rows: `select count(*) from support_docs;`
2. Test Branch B alone — run Supabase RPC with `query_text = 'cancel order'`.
3. Test Branch A — HF embedding returns a 768-length array; Pinecone `/query` returns `matches`.
4. Submit the same Google Form / curl payload as vanilla.
5. In **Executions**, verify: Merge receives both branches → RRF outputs ~10 fused docs → Cohere returns 3 → **Combine Context** outputs **1 item** → LLM Chain runs once → email sent.

If the LLM Chain runs **3 times**, you connected reranked docs directly instead of **Combine Context**.

**Expected hybrid context** — top reranked support responses joined in `Combine Context`:

```json
{
  "query": "How do I cancel my order?",
  "email": "user@example.com",
  "context": "[1] To cancel your purchase...\n\n[2] ...",
  "doc_count": 3
}
```

**Expected retrieval** — documents with full metadata:

```json
{
  "pageContent": "Category: ORDER\nIntent: cancel_order\nQuestion: I need to cancel purchase...",
  "metadata": {
    "category": "ORDER",
    "intent": "cancel_order",
    "response": "To cancel your purchase, please follow these steps..."
  }
}
```

**Wrong retrieval** (re-index if you see this):

```json
{
  "pageContent": "canceling order",
  "metadata": {
    "blobType": "application/json",
    "line": 9,
    "source": "blob"
  }
}
```

---

## Troubleshooting

### CSV extraction fails — "item has no binary data"

**Fix:** Set **Input Binary Field** to the form field name (`upload_csv_file`), not the filename.

### "No Inference Provider available" for nomic-embed-text-v1.5

**Fix:** That model is not hosted on Hugging Face serverless inference. Use `sentence-transformers/all-mpnet-base-v2` (768 dims) instead.

### Groq model not in Embeddings dropdown

**Fix:** Groq chat models (`gpt-oss-120b`, llama, etc.) cannot create embeddings. Use Hugging Face for embeddings; use Groq only for the Chat Model.

### Pinecone returns blob metadata (`blobType`, `line`, `source`)

**Fix:** Default Data Loader was set to **Load All Input Data** or metadata was set incorrectly. Use **Load Specific Data**, map each metadata field as Name/Value pairs, and re-index after clearing the namespace.

### Metadata shows `[object Object]`

**Fix:** Do not put `={{ $json.metadata }}` in the Metadata **Name** field. Add separate properties: Name = `category`, Value = `={{ $json.metadata.category }}`, etc.

### AI Agent answers without searching Pinecone

**Fix:** Strengthen the system prompt and tool description. Try `llama-3.3-70b-versatile` instead of `gpt-oss-120b`. For guaranteed retrieve-first behavior, replace AI Agent with **Question and Answer Chain** + **Vector Store Retriever**.

### Dimension mismatch error

**Fix:** Pinecone index dimensions must match the embedding model. `all-mpnet-base-v2` = **768**. Use the same model at insert and retrieve.

### Workflow timeout on 27K rows

**Fix:** Use **Split in Batches** (100–200) and test with **Limit 500** first.

### Hugging Face embedding URL returns 404 / model not found

**Fix:** Use the router URL: `https://router.huggingface.co/hf-inference/models/sentence-transformers/all-mpnet-base-v2/pipeline/feature-extraction`. Turn **Send Body** ON with `{ "inputs": "your query" }`.

### Pinecone query returns empty or invalid vector

**Fix:** The vector must be a JSON **array** of 768 floats, not a string. Use a Code node to extract `embedding[0]` from the HF response before calling `/query`.

### Supabase RPC `PGRST202` or 404

**Fix:** Create the `search_support_docs` function in Supabase. In the n8n Supabase node, enable **Send Body** and pass `query_text` and `match_count`.

### Hybrid LLM Chain runs multiple times (3 outputs)

**Fix:** Add a **Combine Context** Code node after reranking to merge top docs into a single item before the LLM Chain.

### Cohere rerank receives wrong documents

**Fix:** Reference `$('Code in JavaScript1').all()` (your RRF node name), not `$input`. Use `.first()` / `.all()` — not `.item`.

### IDs in RRF show `\t\n=ORDER_...` prefix

**Fix:** Normalize IDs in the RRF Code node (`replace(/^[\t\n=]+/, '')`). Re-ingest data without `==` double-equals in n8n expressions to prevent `=` prefixes in stored metadata.

### Local file access denied (n8n 2.x)

**Fix:** Set environment variable before starting n8n:

```powershell
$env:N8N_RESTRICT_FILE_ACCESS_TO = "C:\Users\Dell\Desktop\Rag"
n8n start
```

---

## Quick Reference

| Component | Technology | Key |
|-----------|------------|-----|
| Vector DB | Pinecone (768 dim, cosine) | `pinecone_api_key` |
| BM25 / FTS | Supabase PostgreSQL (`support_docs`) | Supabase service role key |
| Embeddings | HF `sentence-transformers/all-mpnet-base-v2` | `huggingface_api_key` |
| Chat LLM | Groq `openai/gpt-oss-120b` | `groq_api_key` |
| Reranker | Cohere | `cohere_api_key` |
| Public webhook | ngrok | `ngrok_api_key` |

| Workflow | Nodes |
|----------|-------|
| **Upsert** | Form → Extract CSV → Code → Pinecone Insert + HF Embeddings + Default Data Loader (+ Supabase for hybrid) |
| **Vanilla query** | Google Form → Apps Script → ngrok → Webhook → AI Agent → Code (parse) → SMTP + Groq + Pinecone + HF Embeddings + Cohere |
| **Hybrid query** | Webhook → Edit Fields → [Pinecone HTTP + Supabase RPC] → Merge → RRF → Cohere → Combine Context → LLM Chain (Groq) → Code → SMTP |

---

## Next Steps

- Replace `{{Order Number}}` and other placeholders with real company values in the Code node
- Add **Window Buffer Memory** for multi-turn support conversations (vanilla AI Agent)
- Monitor Pinecone vector count and re-run ingestion when the CSV is updated
- Consider **Question and Answer Chain** if the AI Agent inconsistently calls the Pinecone tool
- Tune hybrid **RRF K** (default 60) and Cohere **top_n** (default 3) for your query mix
- Export Workflow 4 as JSON and add a download link alongside `RAG Workflow.json`
