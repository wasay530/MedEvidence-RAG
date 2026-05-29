# MedEvidence-RAG

> **Evaluation-driven Retrieval-Augmented Generation system for healthcare and research documents.**

MedEvidence-RAG is a production-oriented RAG platform designed for clinical guidelines, medical research papers, thesis documents, technical reports, and private knowledge bases. It combines document ingestion, multiple chunking strategies, dense and sparse hybrid retrieval, cross-encoder reranking, citation-grounded answer generation, and RAGAS-based evaluation.

This project demonstrates production-ready LLM engineering with a focus on retrieval quality, answer faithfulness, citation grounding, latency, cost, reproducibility, and deployability — not a simple notebook-based chatbot.

---

## Status

> **Current phase: MVP development (Phase 1–2 in progress)**
>
> Benchmark results are not yet available. All TBD values in the benchmark table will be populated after the evaluation pipeline is fully operational. Do not treat this repository as a finished system; treat it as an engineering reference with a clear, measurable roadmap.

---

## Project Summary

MedEvidence-RAG allows users to upload research or healthcare documents and ask natural-language questions. The system retrieves the most relevant evidence chunks, reranks them, generates an answer using an LLM, and returns citations with document names, page numbers, chunk IDs, confidence scores, and latency.

The project includes a full evaluation pipeline to compare retrieval strategies — dense retrieval, BM25, hybrid search, hybrid with reranking, HyDE, and multi-query retrieval — using RAGAS and standard IR metrics.

---

## Why This Project Exists

Most RAG demos stop at uploading PDFs and asking questions. Real-world AI systems require more than that.

A production-relevant RAG system must answer questions such as:

- Which chunking strategy performs best on clinical text?
- Does hybrid search outperform dense-only retrieval?
- Does reranking improve context precision at what latency cost?
- Are generated answers faithful to the retrieved evidence?
- What is the latency and cost per query?
- Can the system return "insufficient evidence" instead of hallucinating?

MedEvidence-RAG is built to answer those questions through a complete backend, retrieval pipeline, and benchmark framework.

---

## Core Capabilities

| Capability | Description |
|---|---|
| Document ingestion | Upload and process PDFs, Markdown, TXT, research papers, guidelines, and reports |
| Text extraction | Extract, clean, normalize, and structure document text with page metadata |
| Chunking strategies | Fixed-size, recursive, and semantic chunking — benchmarkable independently |
| Re-indexing | Re-chunk existing documents with a different strategy without re-uploading |
| Embeddings | OpenAI embeddings and local sentence-transformer embeddings |
| Vector storage | Qdrant for dense vector search with metadata filtering |
| Sparse retrieval | BM25 with serialised index for persistence across restarts |
| Hybrid retrieval | Dense + sparse retrieval with reciprocal rank fusion (configurable k) |
| Query enhancement | Query expansion, HyDE, and multi-query retrieval (each cacheable) |
| Reranking | Cross-encoder reranking for top retrieved candidates |
| Answer generation | LLM-generated responses grounded in retrieved evidence only |
| Structured output | Pydantic-validated generation responses with confidence, citations, and latency |
| Hallucination guard | Structured output validation rejects responses not grounded in retrieved context |
| Citations | Document name, page number, chunk ID, and retrieval score |
| Evaluation | RAGAS metrics, retrieval metrics, latency, and cost tracking |
| Observability | Structured logging and per-request tracing with correlation IDs |
| API-first design | FastAPI backend with Swagger/OpenAPI documentation |
| Developer setup | Docker Compose, `.env.example`, tests, linting, and CI/CD |

---

## System Architecture

![MedEvidence-RAG Architecture](https://github.com/wasay530/MedEvidence-RAG/blob/40a26b6c7be416632dbc99f7e2361fbd1e33cde7/docs/architecture/medevidence_rag_system_architecture.svg)

---

## Architecture Layers

### 1. Data Sources & Input

Supported input formats:

- PDF research papers
- Clinical guidelines
- Thesis documents
- Technical reports
- Markdown files
- Plain text files

Documents are uploaded through the FastAPI backend and processed asynchronously via Celery. The raw extracted text is persisted in PostgreSQL, decoupled from the chunking strategy. This allows re-indexing with any chunking strategy at any time without re-uploading the original file.

---

### 2. Application & Ingestion Layer

**FastAPI backend** exposes REST endpoints for document management, querying, and evaluation. All endpoints are documented via Swagger/OpenAPI at `/docs`.

**Document upload & metadata service** stores document-level metadata, tracks file status, assigns document IDs, and maintains source provenance throughout the pipeline.

**Celery ingestion worker** processes documents asynchronously to prevent upload requests from blocking. The worker is configured with a dead-letter queue and retry policy so that no job is silently lost if Redis restarts mid-processing.

**Text extraction & cleaning** extracts text from PDFs and plain documents, removes malformed characters, normalises whitespace, and preserves page-level metadata.

> **Note on BM25 persistence:** The BM25 index is serialised to disk on build and reloaded on worker startup. This ensures the sparse index survives container restarts without requiring re-ingestion of all documents.

---

### 3. Chunking Layer

The system supports three chunking strategies, each benchmarkable independently.

| Strategy | Description | Use case |
|---|---|---|
| Fixed chunking | Splits text into fixed token or character windows | Simple baseline |
| Recursive chunking | Splits text using paragraph, sentence, and token boundaries | General-purpose RAG |
| Semantic chunking | Splits based on semantic similarity or topic shifts | Research and clinical documents |

Because raw extracted text is stored in PostgreSQL, documents can be re-chunked with a different strategy without re-uploading. This enables fair A/B comparison of chunking strategies on the same source material.

---

### 4. Storage & Indexing Layer

| Storage component | Purpose |
|---|---|
| Qdrant | Dense vector embeddings and metadata filtering (port 6333) |
| BM25 index | Sparse keyword retrieval — serialised to disk, rebuilt on startup |
| PostgreSQL | Raw extracted text, document metadata, query logs, evaluation runs, experiment results |
| Redis | Celery task queue with dead-letter + retry, cache, temporary query/session state (port 6379) |

---

### 5. Query & Retrieval Pipeline

```
User query
→ Query normalisation
→ Optional query expansion
→ Optional HyDE generation    ← LLM call; result cached per query hash to avoid duplicate cost
→ Optional multi-query generation
→ Dense vector retrieval
→ BM25 sparse retrieval
→ Hybrid fusion (RRF, k=60 default, configurable)
→ Cross-encoder reranking
→ Top-k evidence selection
```

**Supported retrieval modes:**

| Mode | Description |
|---|---|
| Dense only | Vector similarity search using embeddings |
| BM25 only | Sparse keyword retrieval |
| Hybrid | Dense + sparse with reciprocal rank fusion |
| Hybrid + rerank | Hybrid retrieval followed by cross-encoder reranking |
| HyDE + hybrid | Hypothetical document embedding before retrieval (cached) |
| Multi-query + hybrid | Multiple reformulated queries combined before ranking |

> **HyDE latency note:** HyDE adds one additional LLM call in the retrieval path. Results are cached by query hash. Benchmark reports break out HyDE latency separately from generation latency so comparison across strategies is fair.

> **RRF tuning:** The reciprocal rank fusion constant `k` defaults to 60 and is exposed via environment variable `RRF_K`. See [Reciprocal Rank Fusion (Cormack et al., 2009)](https://dl.acm.org/doi/10.1145/1571941.1572114) for tuning guidance.

---

### 6. Generation Layer

The generation layer receives the top-k evidence chunks and creates a grounded answer using a structured output schema validated by Pydantic.

The LLM is instructed to:

- Use only retrieved context
- Cite supporting chunks with document name, page, and chunk ID
- Avoid unsupported claims
- Return `"insufficient_evidence"` as the `confidence` field when context is weak
- Provide confidence and latency metadata

A post-generation hallucination guard validates that the returned citations exist in the retrieved set and rejects malformed or uncited responses before they reach the user.

**Example response:**

```json
{
  "answer": "Federated learning preserves privacy by training models locally on client data and sharing model updates rather than raw data.",
  "citations": [
    {
      "document_name": "federated_learning_healthcare.pdf",
      "page": 4,
      "chunk_id": "doc_001_chunk_014",
      "retrieval_score": 0.91
    }
  ],
  "confidence": "high",
  "retrieval_strategy": "hybrid_rerank",
  "latency_ms": 1840,
  "hyde_latency_ms": null,
  "tokens_used": 1204
}
```

---

### 7. Evaluation & Benchmarking

The evaluation pipeline requires a curated ground-truth dataset. The dataset must be built and validated before running benchmarks — RAGAS metrics are meaningless without accurate reference answers and expected evidence.

**Evaluation dataset format:**

```json
{
  "question": "How does federated learning improve privacy in healthcare AI?",
  "reference_answer": "Federated learning keeps raw data on local clients and shares only model updates.",
  "expected_evidence": [
    {
      "document_name": "fl_healthcare_review.pdf",
      "page": 3
    }
  ]
}
```

**Metrics:**

| Metric type | Metrics |
|---|---|
| Retrieval | Recall@k, MRR, Context Precision, Context Recall |
| Generation | Faithfulness, Response Relevancy, Answer Correctness |
| System | Latency (total, retrieval, HyDE, generation), cost per query, token usage |

---

## Tech Stack

### Backend

- Python
- FastAPI
- Pydantic v2 (structured output validation)
- Uvicorn
- Celery
- Redis
- PostgreSQL
- SQLAlchemy

### Retrieval & LLM

- OpenAI API
- Sentence Transformers
- Qdrant
- BM25 (serialised index)
- Cross-encoder reranker
- RAGAS
- Transformers
- scikit-learn

### Frontend & Developer Tools

- Streamlit demo UI
- Docker
- Docker Compose
- GitHub Actions
- Pytest
- Ruff
- Pre-commit
- Swagger/OpenAPI

---

## Repository Structure

```text
med-evidence-rag/
│
├── app/
│   ├── main.py
│   ├── api/
│   │   ├── routes_documents.py
│   │   ├── routes_query.py
│   │   └── routes_evaluation.py
│   │
│   ├── core/
│   │   ├── config.py           # all env vars including RRF_K, DEFAULT_TOP_K, RERANK_TOP_K
│   │   ├── logging.py          # structured JSON logging with correlation IDs
│   │   ├── tracing.py          # per-request trace context
│   │   └── security.py
│   │
│   ├── ingestion/
│   │   ├── pdf_loader.py
│   │   ├── text_cleaner.py
│   │   ├── chunkers.py
│   │   └── metadata.py
│   │
│   ├── embeddings/
│   │   ├── openai_embedder.py
│   │   └── sentence_transformer_embedder.py
│   │
│   ├── retrieval/
│   │   ├── dense_retriever.py
│   │   ├── bm25_retriever.py   # includes serialise/load for index persistence
│   │   ├── hybrid_retriever.py # RRF with configurable k
│   │   ├── hyde.py             # result cached by query hash
│   │   ├── multi_query.py
│   │   └── reranker.py
│   │
│   ├── generation/
│   │   ├── prompt_templates.py
│   │   ├── answer_generator.py
│   │   └── hallucination_guard.py   # validates citations against retrieved set
│   │
│   ├── evaluation/
│   │   ├── ragas_eval.py
│   │   ├── benchmark_runner.py
│   │   └── report_writer.py
│   │
│   ├── workers/
│   │   └── celery_worker.py    # includes dead-letter queue and retry config
│   │
│   └── db/
│       ├── models.py           # includes raw_text column on Document model
│       └── session.py
│
├── ui/
│   └── streamlit_app.py
│
├── docs/
│   └── architecture/
│       └── medevidence-rag-architecture.png
│
├── tests/
│   ├── test_chunking.py
│   ├── test_retrieval.py
│   ├── test_generation.py       # includes hallucination guard tests
│   └── test_api.py
│
├── data/
│   ├── sample_docs/
│   └── eval_sets/
│       └── healthcare_rag_eval.json   # must be populated before running benchmarks
│
├── reports/
│   ├── retrieval_benchmark.md
│   └── ragas_results.json
│
├── docker-compose.yml
├── Dockerfile
├── Makefile
├── requirements.txt
├── .env.example
├── README.md
└── LICENSE
```

---

## API Overview

### Document APIs

```http
POST   /documents/upload
GET    /documents
GET    /documents/{document_id}
DELETE /documents/{document_id}
POST   /documents/{document_id}/reindex    # re-chunk with a different strategy
```

### Query APIs

```http
POST /query
POST /query/dense
POST /query/bm25
POST /query/hybrid
POST /query/hyde
POST /query/multi-query
```

### Evaluation APIs

```http
POST /evaluate/run
GET  /evaluate/results
GET  /evaluate/report
```

---

## Example Query Request

```json
{
  "question": "How does federated learning preserve privacy in healthcare AI?",
  "retrieval_strategy": "hybrid_rerank",
  "top_k": 5,
  "document_ids": ["doc_001", "doc_002"]
}
```

---

## Example Query Response

```json
{
  "answer": "Federated learning preserves privacy by keeping raw data on local clients and sharing model updates instead of centralising sensitive data.",
  "citations": [
    {
      "document_name": "federated_learning_healthcare.pdf",
      "page": 4,
      "chunk_id": "doc_001_chunk_014",
      "retrieval_score": 0.91
    }
  ],
  "confidence": "high",
  "retrieval_strategy": "hybrid_rerank",
  "latency_ms": 1840,
  "hyde_latency_ms": null,
  "tokens_used": 1204
}
```

---

## Local Setup

### 1. Clone the repository

```bash
git clone https://github.com/wasay530/MedEvidence-RAG.git
cd MedEvidence-RAG
```

### 2. Create environment file

```bash
cp .env.example .env
```

Update the required variables:

```env
OPENAI_API_KEY=your_openai_api_key
DATABASE_URL=postgresql://postgres:postgres@postgres:5432/medevidence
REDIS_URL=redis://redis:6379/0
QDRANT_URL=http://qdrant:6333
EMBEDDING_PROVIDER=openai
LLM_PROVIDER=openai
DEFAULT_TOP_K=20
RERANK_TOP_K=5
RRF_K=60
LOG_LEVEL=INFO
```

### 3. Start services

```bash
docker compose up --build
```

### 4. Open the API docs

```
http://localhost:8000/docs
```

### 5. Open the Streamlit demo

```
http://localhost:8501
```

---

## Running Without Docker

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

Start Celery worker:

```bash
celery -A app.workers.celery_worker worker --loglevel=info
```

Start Streamlit UI:

```bash
streamlit run ui/streamlit_app.py
```

---

## Running Tests

```bash
pytest
```

Run linting:

```bash
ruff check .
```

Run formatting:

```bash
ruff format .
```

---

## Running Evaluation

> Before running evaluation, populate `data/eval_sets/healthcare_rag_eval.json` with ground-truth question/answer/evidence triples relevant to your document set. Benchmark scores are meaningless without a validated dataset.

```bash
python -m app.evaluation.benchmark_runner \
  --dataset data/eval_sets/healthcare_rag_eval.json \
  --strategies dense bm25 hybrid hybrid_rerank hyde_hybrid_rerank \
  --output reports/ragas_results.json
```

Generate Markdown report:

```bash
python -m app.evaluation.report_writer \
  --input reports/ragas_results.json \
  --output reports/retrieval_benchmark.md
```

---

## Docker Services

| Service | Purpose |
|---|---|
| `api` | FastAPI backend (port 8000) |
| `worker` | Celery ingestion worker with dead-letter queue |
| `redis` | Queue broker and cache (port 6379) |
| `postgres` | Metadata, raw text, and experiment database |
| `qdrant` | Vector database (port 6333) |
| `streamlit` | Demo UI (port 8501) |

---

## Environment Variables

| Variable | Description |
|---|---|
| `OPENAI_API_KEY` | API key for OpenAI models |
| `DATABASE_URL` | PostgreSQL connection URL |
| `REDIS_URL` | Redis connection URL |
| `QDRANT_URL` | Qdrant service URL |
| `EMBEDDING_PROVIDER` | `openai` or `sentence_transformers` |
| `LLM_PROVIDER` | LLM provider name |
| `DEFAULT_TOP_K` | Number of chunks retrieved before reranking (default: 20) |
| `RERANK_TOP_K` | Number of chunks kept after reranking (default: 5) |
| `RRF_K` | Reciprocal rank fusion constant (default: 60) |
| `LOG_LEVEL` | Logging level (default: INFO) |

---

## Known Limitations & Design Decisions

| Area | Decision | Rationale |
|---|---|---|
| Authentication | Not yet implemented | Planned for Phase 4; do not expose this service publicly until complete |
| BM25 persistence | Serialised to disk, rebuilt on startup | rank-bm25 is in-memory only; serialisation prevents data loss on restart |
| HyDE caching | Results cached by query hash | HyDE doubles per-query LLM cost; caching prevents duplicate calls in evaluation loops |
| RRF constant | Configurable via `RRF_K` env var (default 60) | Cormack et al. recommend 60 as a robust default; expose for ablation studies |
| Ground-truth dataset | Not included | Domain-specific medical QA annotation requires expert review; a sample template is provided |
| Rate limiting | Not yet implemented | Planned for Phase 4 |
| Distributed tracing | Correlation IDs on every request | Full distributed tracing (OpenTelemetry) planned for Phase 4 |

---

## Roadmap

### Phase 1 — MVP RAG backend

- [ ] FastAPI application setup
- [ ] PDF upload endpoint
- [ ] Text extraction pipeline with raw text persistence
- [ ] Fixed and recursive chunking
- [ ] Re-indexing endpoint (re-chunk without re-upload)
- [ ] OpenAI embeddings
- [ ] Qdrant vector indexing
- [ ] Basic citation-grounded query endpoint
- [ ] Pydantic-validated structured output
- [ ] Hallucination guard on generation output
- [ ] Structured JSON logging with correlation IDs
- [ ] Docker Compose setup

### Phase 2 — Advanced retrieval

- [ ] BM25 sparse retrieval with serialised index persistence
- [ ] Hybrid retrieval with configurable RRF (k exposed via env)
- [ ] Cross-encoder reranking
- [ ] Query expansion
- [ ] HyDE retrieval with per-query caching
- [ ] Multi-query retrieval

### Phase 3 — Evaluation

- [ ] Ground-truth evaluation dataset (healthcare domain)
- [ ] RAGAS integration
- [ ] Retrieval metrics (Recall@k, MRR, Context Precision, Context Recall)
- [ ] Generation metrics (Faithfulness, Response Relevancy, Answer Correctness)
- [ ] Per-strategy latency breakdown (retrieval vs HyDE vs generation)
- [ ] Benchmark report generation
- [ ] README benchmark table populated with real numbers

### Phase 4 — Production hardening

- [ ] Authentication and authorisation
- [ ] Rate limiting
- [ ] OpenTelemetry distributed tracing
- [ ] Cost tracking dashboard
- [ ] CI/CD pipeline with staging environment
- [ ] Deployment guide (cloud and on-premise)

---

## Example Use Cases

- Healthcare literature question answering
- Clinical guideline search
- Research paper summarisation
- Thesis and technical report exploration
- Private document Q&A
- Evidence-grounded academic assistant
- RAG evaluation benchmarking

---

## What Makes This Project Different

MedEvidence-RAG is not a chatbot over PDFs.

It is designed to show:

1. **Retrieval engineering** through dense, sparse, hybrid, and reranked search with configurable, tunable parameters.
2. **Evaluation discipline** through RAGAS and IR metrics applied to a validated ground-truth dataset.
3. **Production mindset** through FastAPI, Docker, Celery with dead-letter queues, BM25 index persistence, and structured logging.
4. **Healthcare/research relevance** through evidence-grounded answers, citation-first design, and a hallucination guard.
5. **Reproducibility** through re-indexing support, configurable RRF, and separated latency reporting per pipeline stage.
6. **Portfolio strength** through measurable benchmark results — not vague claims.

---

## Suggested GitHub Topics

```
rag
llm
fastapi
qdrant
ragas
hybrid-search
bm25
reranking
healthcare-ai
retrieval-augmented-generation
openai
sentence-transformers
docker
ai-engineering
clinical-ai
```

---

## License

This project is released under the MIT License.

---

## Author

**Abdul Wasay Sardar**  
ML & AI Engineer | Software Engineer | Researcher

This project connects backend production engineering with applied AI research in healthcare, NLP, and privacy-preserving machine learning.
