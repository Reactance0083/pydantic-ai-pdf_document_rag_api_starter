# PDF/Document RAG API Starter

A production-ready FastAPI service for building PDF/document RAG (Retrieval-Augmented Generation) systems. Ingest PDFs, embed them into pgvector, and query with typed answers and page-level citations. Built on **pydantic-ai** for structured outputs and **FastAPI** for the HTTP layer.

## Overview

Stop wrestling with notebook-only LangChain tutorials. This template gives you:

- **FastAPI** service with async endpoints
- **pydantic-ai** agent returning validated answers with citations
- **pgvector** for embedding storage and similarity search
- **PyMuPDF** for fast PDF text extraction with page tracking
- **SSE streaming** endpoint for token-by-token responses
- **Streamlit demo client** (~50 lines) for instant UI testing
- Typed citation objects (`{document_id, page, snippet, score}`)
- Docker Compose for one-command Postgres setup

Ships with sensible defaults: OpenAI embeddings (`text-embedding-3-small`), GPT-4o-mini for generation, 800-token chunks with 100-token overlap. Swap any of these via `.env`.

## What it does

1. **Ingest**: Upload a PDF → extract text per page → chunk → embed → store in pgvector
2. **Query**: POST a question → semantic search → pydantic-ai agent generates a typed answer with page-level citations
3. **Stream**: Same as query but as Server-Sent Events for token streaming
4. **List/Delete**: Manage ingested documents

## Prerequisites

- Python 3.11+
- Docker & Docker Compose (for Postgres + pgvector)
- An OpenAI API key (or any pydantic-ai-supported provider)
- `uv` or `pip` for dependency installation

## Setup

1. **Clone and enter the repo**
   ```bash
   git clone https://github.com/yourname/pdf-rag-starter.git
   cd pdf-rag-starter
   ```

2. **Copy and edit environment variables**
   ```bash
   cp .env.example .env
   # Edit .env and set OPENAI_API_KEY
   ```

3. **Start Postgres with pgvector**
   ```bash
   docker compose up -d db
   ```
   This launches `ankane/pgvector:latest` on port `5432` with the database `ragdb`.

4. **Install Python dependencies**
   ```bash
   uv venv && source .venv/bin/activate
   uv pip install -r requirements.txt
   ```
   Or with pip:
   ```bash
   python -m venv .venv && source .venv/bin/activate
   pip install -r requirements.txt
   ```

5. **Run database migrations**
   ```bash
   python -m app.db.migrate
   ```
   Creates the `documents`, `chunks` tables and the `vector` extension.

6. **Start the API**
   ```bash
   uvicorn app.main:app --reload --port 8000
   ```
   Visit `http://localhost:8000/docs` for the Swagger UI.

7. **(Optional) Launch the Streamlit demo**
   ```bash
   streamlit run demo/streamlit_app.py
   ```

Total time: ~5 minutes if Docker is already installed.

## Usage

### Ingest a PDF

```bash
curl -X POST http://localhost:8000/ingest \
  -F "file=@./samples/handbook.pdf" \
  -F "title=Employee Handbook"
```

Response:
```json
{
  "document_id": "8e2a...c9",
  "title": "Employee Handbook",
  "pages": 42,
  "chunks": 187,
  "status": "indexed"
}
```

### Ask a question

```bash
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{"question": "What is the vacation policy?", "top_k": 5}'
```

Response:
```json
{
  "answer": "Full-time employees receive 20 days of paid vacation per year, accrued monthly.",
  "citations": [
    {
      "document_id": "8e2a...c9",
      "page": 12,
      "snippet": "All full-time employees accrue 1.67 days of vacation per month...",
      "score": 0.89
    }
  ],
  "confidence": 0.91
}
```

### Stream an answer (SSE)

```bash
curl -N -X POST http://localhost:8000/query/stream \
  -H "Content-Type: application/json" \
  -d '{"question": "Summarize section 3.", "top_k": 5}'
```

With Python `httpx`:
```python
import httpx, json

with httpx.stream("POST", "http://localhost:8000/query/stream",
                 json={"question": "Summarize section 3.", "top_k": 5},
                 timeout=None) as r:
    for line in r.iter_lines():
        if line.startswith("data: "):
            print(json.loads(line[6:]))
```

### List documents

```bash
curl http://localhost:8000/documents
```

### Delete a document

```bash
curl -X DELETE http://localhost:8000/documents/8e2a...c9
```

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/ingest` | Upload PDF (multipart). Returns `document_id`, chunk count. |
| `POST` | `/query` | Body: `{question, top_k?, document_ids?}`. Returns typed answer + citations. |
| `POST` | `/query/stream` | Same body. Returns SSE stream of tokens, then final citations. |
| `GET`  | `/documents` | List all ingested documents. |
| `GET`  | `/documents/{id}` | Get document metadata and chunk count. |
| `DELETE` | `/documents/{id}` | Delete document and its chunks. |
| `GET`  | `/health` | Liveness check. Returns `{"status": "ok"}`. |

Full schemas at `http://localhost:8000/docs`.

## Configuration (.env variables)

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENAI_API_KEY` | — | Required. Your OpenAI key. |
| `LLM_MODEL` | `openai:gpt-4o-mini` | pydantic-ai model string. |
| `EMBEDDING_MODEL` | `text-embedding-3-small` | OpenAI embedding model. |
| `EMBEDDING_DIM` | `1536` | Must match the embedding model output dim. |
| `DATABASE_URL` | `postgresql+asyncpg://rag:rag@localhost:5432/ragdb` | Async Postgres URL. |
| `CHUNK_SIZE` | `800` | Target tokens per chunk. |
| `CHUNK_OVERLAP` | `100` | Token overlap between chunks. |
| `TOP_K_DEFAULT` | `5` | Default number of chunks retrieved per query. |
| `MAX_UPLOAD_MB` | `25` | Max PDF upload size. |
| `CORS_ORIGINS` | `*` | Comma-separated allowed origins. |
| `LOG_LEVEL` | `INFO` | `DEBUG`, `INFO`, `WARNING`, `ERROR`. |

## Customization

**Swap the embedding provider**
Edit `app/services/embeddings.py`. The interface is one async function: `embed(texts: list[str]) -> list[list[float]]`. Drop in Cohere, Voyage, or a local model — just update `EMBEDDING_DIM` and re-run migrations.

**Use a different LLM**
Change `LLM_MODEL` to any [pydantic-ai-supported model](https://ai.pydantic.dev/models/), e.g. `anthropic:claude-3-5-sonnet-latest` or `groq:llama-3.3-70b-versatile`. Add the relevant API key to `.env`.

**Tune chunking**
Edit `app/services/chunker.py`. The default splitter is recursive-character with token-based sizing via `tiktoken`. For tables/code-heavy PDFs, try semantic chunking by overriding `chunk_text()`.

**Change the answer schema**
The pydantic-ai agent's output type lives in `app/agents/rag_agent.py`:
```python
class RAGAnswer(BaseModel):
    answer: str
    citations: list[Citation]
    confidence: float
```
Add fields (e.g., `follow_up_questions: list[str]`) and the agent will populate them — validated automatically.

**Add auth**
Drop an API-key dependency into `app/deps.py` and apply it via `dependencies=[Depends(api_key_auth)]` on the router.

**Support other file types**
`app/services/loader.py` dispatches by MIME type. Add handlers for `.docx` (python-docx), `.html` (BeautifulSoup), or `.md` and they'll flow through the same chunking + embedding pipeline.

## License

MIT. Use it, ship it, sell it. No attribution required.