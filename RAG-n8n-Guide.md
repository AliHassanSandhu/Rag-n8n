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
│  Webhook → AI Agent                                          │
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

### Step 8: Webhook Trigger

Add a **Webhook** node (POST). Use this URL to send questions from external apps or Google Forms.

Example request body:

```json
{
  "question": "How do I cancel my order?"
}
```

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

Local n8n webhooks are only reachable on your machine. Use **ngrok** to expose them publicly (e.g. for Google Forms or external apps).

1. Install ngrok and add your authtoken (`ngrok_api_key`).
2. Start n8n (default port `5678`).
3. Run:

```bash
ngrok http 5678
```

4. Copy the HTTPS URL (e.g. `https://abc123.ngrok-free.app`).
5. In n8n, open your Webhook node and use:

```
https://abc123.ngrok-free.app/webhook/<your-webhook-path>
```

Use this URL in Google Forms, Postman, or any external trigger.

---

## Testing

### Ingestion

1. Upload CSV via Form Trigger.
2. Run with **Limit 50** first.
3. Confirm ~50 vectors in Pinecone with correct metadata (including `response`).
4. Remove Limit, run full dataset with Split in Batches.

### Query

Send a POST to your webhook:

```json
{
  "question": "How do I cancel my order?"
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
| **Query** | Webhook → AI Agent + Groq Chat + Pinecone Retrieve Tool + HF Embeddings + Cohere Reranker |

---

## Next Steps

- Replace `{{Order Number}}` and other placeholders with real company values in the Code node
- Add **Window Buffer Memory** for multi-turn support conversations
- Monitor Pinecone vector count and re-run ingestion when the CSV is updated
- Consider **Question and Answer Chain** if the AI Agent inconsistently calls the Pinecone tool
