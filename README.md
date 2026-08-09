# n8n Telegram RAG Agent

A Telegram chatbot, built entirely in n8n, that answers questions about your own documents using **RAG (Retrieval-Augmented Generation)** with a self-hosted vector database — and a **100% local LLM stack (Ollama)**, so no document content, question, or conversation ever leaves infrastructure you control.

This is an evolution of [n8n-local-pdf-agent](https://github.com/SantiagoE-n/n8n-local-pdf-agent): that project fed an entire PDF into the prompt on every request, which doesn't scale past small documents. This one splits documents into chunks, embeds them, and retrieves only the relevant pieces per question — the standard production pattern for document Q&A.

## The problem

Feeding a whole document into an LLM's context window works for a one-page CV, but breaks down fast: bigger documents get truncated, costs and latency grow, and the model's attention gets diluted across irrelevant sections. RAG solves this by searching for the *specific* relevant fragments first, and only passing those to the model — plus it lets you query a document that would never fit in a context window at all.

## How it works

The project is split into **two separate n8n workflows**, which is the standard RAG pattern — one to ingest documents, one to answer questions:

### 1. Ingestion workflow (`ingestion-workflow.json`)

Run manually whenever you want to add a document to the knowledge base:

```
[Manual Trigger] → [Read file from disk] → [Extract PDF text]
   → [Recursive Character Text Splitter (chunk: 1000, overlap: 200)]
   → [Default Data Loader] → [Qdrant Vector Store: Insert]
                                       ↑
                          [Ollama Embeddings: nomic-embed-text]
```

The document gets split into overlapping chunks, each chunk is converted into a vector via a local embedding model, and stored in a Qdrant collection.

### 2. Telegram chat workflow (`telegram-chat-workflow.json`)

Runs continuously, listening for messages:

```
[Telegram Trigger] → [Extract message + chat ID] → [AI Agent] → [Reply on Telegram]
                                                        ├── Chat Model: Ollama (granite4.1:3b)
                                                        ├── Memory: per-user, keyed by Telegram chat ID
                                                        └── Tool: Qdrant Vector Store (retrieve-as-tool)
                                                                       ↑
                                                          Ollama Embeddings: nomic-embed-text
```

The AI Agent decides, per message, whether it needs to search the knowledge base — when it does, it queries Qdrant for the most semantically relevant chunks and grounds its answer in those, instead of the whole document.

## Technical decisions

**Two workflows, not one.** Ingestion and querying have completely different triggers, frequencies, and failure modes — mixing them into a single workflow would make both harder to reason about and to fix independently.

**Local models throughout (Ollama).** Chat model: `granite4.1:3b`, chosen specifically because it supports tool calling reliably at a small size — most local models under ~7B don't support tool calling at all, or fail with more than one tool. Embedding model: `nomic-embed-text`, used identically at ingestion and query time (mixing embedding models between insert and retrieve breaks similarity search).

**Per-user memory, keyed by Telegram `chat.id`.** Each conversation is isolated — one user's chat history never leaks into another's.

**Chunking with overlap** (1000 characters, 200 overlap) keeps each vector focused on one coherent idea while avoiding losing context right at chunk boundaries.

**Error handling on the user-facing workflow.** The AI Agent node uses `continueErrorOutput`: if the model, memory, or vector search fails, the user gets a clear "something went wrong, try again" message on Telegram instead of silence. The ingestion workflow doesn't have this — it's a manually-run admin tool, not something with a waiting caller, so failures are visible directly in n8n's execution log.

## Known limitations / next steps

1. **Ingestion path is hardcoded.** The file to ingest is a fixed path in the "Read/Write Files from Disk" node — adding a new document means editing that node manually rather than passing a filename dynamically.
2. **No deduplication between ingestion runs.** Re-ingesting a new document into the same Qdrant collection doesn't remove the old one — both sets of vectors coexist, which can blur retrieval results if documents overlap in topic.
3. **Telegram webhook requires a public HTTPS tunnel (ngrok) for local development.** This is fine for a demo but not a durable setup — running this in production would mean deploying n8n on a server with a real domain and SSL certificate.
4. **No collection-per-document or metadata filtering.** All chunks from all ingested documents currently live in a single Qdrant collection with no way to scope a search to one specific document.
5. **Small chat model.** `granite4.1:3b` is chosen for speed and tool-calling reliability over raw reasoning quality; a larger local model would likely produce better answers on complex questions, at the cost of speed.

## Stack

n8n · Telegram Trigger/Bot API · Qdrant (self-hosted, vector database) · Ollama (local LLM + embeddings) · LangChain Agent (via n8n) · Recursive Character Text Splitter · Buffer Window Memory

## How to run it

**Requirements:** Docker, Ollama running locally with `granite4.1:3b` and `nomic-embed-text` pulled, Qdrant running (`docker run -p 6333:6333 qdrant/qdrant`), and a Telegram bot token from [@BotFather](https://t.me/BotFather).

1. Import both `ingestion-workflow.json` and `telegram-chat-workflow.json` into n8n.
2. Set up credentials: Ollama (`http://localhost:11434` or `http://host.docker.internal:11434` if n8n runs in Docker), Qdrant (`http://localhost:6333`, no API key needed for local/unauthenticated instances), Telegram Bot API (token from BotFather).
3. Place a PDF at the path configured in the ingestion workflow's "Read/Write Files from Disk" node, and run it manually once (**Execute Workflow**).
4. Since Telegram requires a public HTTPS webhook, expose your local n8n instance with a tunnel (e.g. `ngrok http 5678`), set n8n's `WEBHOOK_URL` environment variable to that HTTPS URL, and restart n8n.
5. Activate the Telegram chat workflow.
6. Message your bot with a question about the ingested document.

## Screenshots

_Add a screenshot of both workflow canvases here (`assets/ingestion-workflow.png`, `assets/telegram-chat-workflow.png`)._
