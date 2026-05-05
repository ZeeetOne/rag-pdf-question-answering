# RAG Production App

An end-to-end **Retrieval-Augmented Generation (RAG)** system built with Python. Upload PDFs, ask questions in natural language, and get AI-powered answers grounded in your documents — with semantic search, event-driven workflows, and production-grade controls.

---

## Features

- **PDF Ingestion** — Upload PDFs via a web UI; documents are chunked, embedded, and indexed automatically
- **Semantic Search** — Vector similarity search using Qdrant to retrieve the most relevant document chunks
- **AI-Powered Q&A** — GPT-4o-mini synthesizes answers from retrieved context with source attribution
- **Event-Driven Workflows** — Inngest orchestrates ingestion and query pipelines with retry and step isolation
- **Rate Limiting & Concurrency Controls** — Per-user throttling (2 req/min), global concurrency cap (10), and cooldown windows (1 per 4 hours)
- **Streamlit Frontend** — Clean, interactive UI for uploading PDFs and querying with configurable result count

---

## Architecture

```
┌─────────────────┐        Events        ┌──────────────────────┐
│  Streamlit UI   │ ──────────────────► │  Inngest Workflows   │
│  (Frontend)     │ ◄────────────────── │  (Orchestration)     │
└─────────────────┘     Poll Results    └──────────┬───────────┘
                                                   │
                              ┌────────────────────┼────────────────────┐
                              │                    │                    │
                    ┌─────────▼──────┐  ┌──────────▼──────┐  ┌────────▼────────┐
                    │  data_loader   │  │   vector_db     │  │   OpenAI API    │
                    │  PDF Chunking  │  │  Qdrant Search  │  │  Embeddings +   │
                    │  + Embeddings  │  │  + Upsert       │  │  GPT-4o-mini    │
                    └────────────────┘  └─────────────────┘  └─────────────────┘
```

### Workflow — Ingest

1. User uploads a PDF via the Streamlit UI
2. Streamlit fires a `rag/ingest_pdf` event to Inngest
3. Inngest triggers the ingest function: load → chunk → embed → upsert to Qdrant

### Workflow — Query

1. User submits a question with a `top_k` chunk count
2. Streamlit fires a `rag/query_pdf_ai` event to Inngest
3. Inngest triggers the query function: embed query → search Qdrant → call GPT-4o-mini
4. UI polls the Inngest API until the run completes, then displays the answer and sources

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Streamlit |
| Backend / API | FastAPI + Uvicorn |
| Workflow Orchestration | Inngest |
| Vector Database | Qdrant |
| Embeddings | OpenAI `text-embedding-3-large` (3072 dims) |
| LLM | OpenAI GPT-4o-mini |
| PDF Processing | LlamaIndex (PDFReader + SentenceSplitter) |
| Package Manager | uv |

---

## Setup

### Prerequisites

- Python 3.12+
- [uv](https://docs.astral.sh/uv/) package manager
- [Qdrant](https://qdrant.tech/documentation/quick-start/) running locally
- [Inngest Dev Server](https://www.inngest.com/docs/dev-server)
- OpenAI API key

### 1. Clone and install dependencies

```bash
git clone https://github.com/ZeeetOne/RAGProductionApp.git
cd RAGProductionApp
uv sync
```

### 2. Configure environment variables

```bash
cp .env.example .env
```

Edit `.env`:

```env
OPENAI_API_KEY=your_openai_api_key_here
```

### 3. Start Qdrant

```bash
docker run -p 6333:6333 qdrant/qdrant
```

### 4. Start the Inngest Dev Server

```bash
npx inngest-cli@latest dev
```

### 5. Start the FastAPI backend

```bash
uv run uvicorn main:app --reload
```

### 6. Start the Streamlit frontend

```bash
uv run streamlit run streamlit_app.py
```

Open [http://localhost:8501](http://localhost:8501) in your browser.

---

## Usage

### Upload a PDF

1. Navigate to the **Upload PDF** section
2. Select a PDF file from your machine
3. Click **Upload & Ingest** — the document will be chunked, embedded, and stored in Qdrant

### Ask a Question

1. Enter your question in the **Query** section
2. Adjust **Top K** to control how many chunks are retrieved (1–20)
3. Click **Submit** — the app polls for results and displays the AI answer with source references

---

## Project Structure

```
RAGProductionApp/
├── main.py              # FastAPI app + Inngest workflow functions
├── streamlit_app.py     # Streamlit frontend UI
├── vector_db.py         # Qdrant client wrapper (upsert + search)
├── data_loader.py       # PDF chunking and OpenAI embedding logic
├── custom_types.py      # Pydantic models for workflow step I/O
├── pyproject.toml       # Project metadata and dependencies
├── .env.example         # Environment variable template
└── uploads/             # Uploaded PDFs (git-ignored)
```

---

## Rate Limiting & Concurrency

The Inngest workflows are configured with production-grade controls:

| Control | Value | Scope |
|---|---|---|
| Max Concurrency | 10 simultaneous runs | Global |
| Throttle | 2 requests per minute | Per source |
| Rate Limit | 1 request per 4 hours | Per source |

---

## License

MIT
