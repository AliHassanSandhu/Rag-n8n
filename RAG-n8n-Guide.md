# Building a RAG Customer Support Bot in n8n

A step-by-step guide to building a **Retrieval-Augmented Generation (RAG)** pipeline in n8n using the Bitext Customer Support dataset, Pinecone, Hugging Face embeddings, Groq chat, and Cohere reranking.

> **Viewing this guide:** Workflow screenshots are **embedded inline** (base64 data URLs) so they render in online Markdown viewers without local file paths. If your platform blocks embedded images (e.g. GitHub), push the `screenshots/` folder to a repo and use hosted URLs such as `https://raw.githubusercontent.com/<user>/<repo>/main/screenshots/upload-and-process-csv.png`.

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
9. [Workflow 3 — RAG Chat Pipeline](#workflow-3--rag-chat-pipeline)
10. [Expose Your Workflow with ngrok](#expose-your-workflow-with-ngrok)
11. [Testing](#testing)
12. [Troubleshooting](#troubleshooting)

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

This guide uses **two separate workflows**:

| Workflow | Purpose |
|----------|---------|
| **Ingestion (upsert)** | Load CSV → embed → store in Pinecone (run once or on schedule) |
| **Chat (query)** | Receive user question → search Pinecone → generate answer |

---

## Prerequisites

### Software

- **n8n** installed locally (`n8n start` or Docker)
- **Node.js** (if running n8n via npm)
- **ngrok** (optional — to expose local webhooks to the internet)

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

A ready-made n8n workflow export (`RAG Workflow.json`) is available for download. Import it into n8n to get the ingestion and chat pipelines pre-wired, then attach your own credentials.

**[Download RAG Workflow.json (Google Drive)](https://drive.google.com/file/d/1dJ6OdfAlFp8ziWUHVAIfPEUhlb0ExvUU/view?usp=sharing)**

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
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   CHAT WORKFLOW (query)                      │
│                                                              │
│  Google Form → Apps Script → ngrok → Webhook → AI Agent       │
│               ├── Groq Chat Model                            │
│               └── Pinecone (Retrieve Tool)                   │
│                        ├── HF Embeddings (same model)        │
│                        └── Cohere Reranker                   │
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

---

## Workflow 3 — RAG Chat Pipeline

This workflow answers user questions by searching Pinecone and generating a response with Groq.

![RAG and Chat workflow with Groq, Pinecone, Hugging Face, and Cohere](https://github.com/AliHassanSandhu/Rag-n8n/blob/main/screenshots/rag-and-chat-workflow.png)

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
2. Add a **Short answer** question for the user's support question, e.g. title: **`Question`** or **`Your support question`**.
3. (Optional) Add more fields (name, email, category).
4. Click **Send** → link icon → copy the form URL for testers.

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

```
You are a customer support assistant.

When a user asks a question:
1. Always search the knowledge base tool first.
2. Answer ONLY using the retrieved support responses (metadata.response).
3. Be concise and friendly.
4. If nothing relevant is found, say you don't know and offer human support.
5. Do not invent policies, phone numbers, or URLs.
```

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

### Canvas Summary

```
Webhook → AI Agent
              ├── Chat Model      → Groq Chat Model
              ├── Tool            → Pinecone (Retrieve as Tool)
              │                         ├── Embeddings HF (same model)
              │                         └── Cohere Reranker
              └── Memory            → Window Buffer Memory (optional)
```

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

### Query

**Option 1 — Google Form:** Submit the form with a question like *How do I cancel my order?* (see [Step 8.6 — Test the webhook end-to-end](#86--test-the-webhook-end-to-end)).

**Option 2 — curl / Postman:** Send a POST to your ngrok webhook URL with the same JSON shape Apps Script sends (keys = form question titles):

```json
{
  "Question": "How do I cancel my order?"
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
| Embeddings | HF `sentence-transformers/all-mpnet-base-v2` | `huggingface_api_key` |
| Chat LLM | Groq `openai/gpt-oss-120b` | `groq_api_key` |
| Reranker | Cohere | `cohere_api_key` |
| Public webhook | ngrok | `ngrok_api_key` |

| Workflow | Nodes |
|----------|-------|
| **Upsert** | Form → Extract CSV → Code → Pinecone Insert + HF Embeddings + Default Data Loader |
| **Query** | Google Form → Apps Script → ngrok → Webhook → AI Agent + Groq Chat + Pinecone Retrieve Tool + HF Embeddings + Cohere Reranker |

---

## Next Steps

- Replace `{{Order Number}}` and other placeholders with real company values in the Code node
- Add **Window Buffer Memory** for multi-turn support conversations
- Monitor Pinecone vector count and re-run ingestion when the CSV is updated
- Consider **Question and Answer Chain** if the AI Agent inconsistently calls the Pinecone tool
