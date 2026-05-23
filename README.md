# MedEvidence-RAG

> **Evaluation-driven Retrieval-Augmented Generation system for healthcare and research documents.**

MedEvidence-RAG is a production-oriented RAG platform designed for clinical guidelines, medical research papers, thesis documents, technical reports, and private knowledge bases. It combines document ingestion, multiple chunking strategies, dense and sparse hybrid retrieval, cross-encoder reranking, citation-grounded answer generation, and RAGAS-based evaluation.

This project is designed to demonstrate production-ready LLM engineering rather than a simple notebook-based chatbot. It focuses on retrieval quality, answer faithfulness, citation grounding, latency, cost, reproducibility, and deployability.

---

## Project Summary

**MedEvidence-RAG** allows users to upload research or healthcare documents and ask natural-language questions. The system retrieves the most relevant evidence chunks, reranks them, generates an answer using an LLM, and returns citations with document names, page numbers, chunk IDs, confidence, and latency.

The project also includes an evaluation pipeline to compare retrieval strategies such as dense retrieval, BM25, hybrid search, hybrid search with reranking, HyDE, and multi-query retrieval.

---

## Why This Project Exists

Most RAG demos stop at uploading PDFs and asking questions. Real-world AI systems require more than that.

A production-relevant RAG system must answer questions such as:

* Which chunking strategy performs best?
* Does hybrid search outperform dense-only retrieval?
* Does reranking improve context precision?
* Are generated answers faithful to the retrieved evidence?
* What is the latency and cost per query?
* Can the system say “insufficient evidence” instead of hallucinating?

MedEvidence-RAG is built to answer those questions through a complete backend, retrieval pipeline, and benchmark framework.

---

## Core Capabilities

| Capability          | Description                                                                      |
| ------------------- | -------------------------------------------------------------------------------- |
| Document ingestion  | Upload and process PDFs, Markdown, TXT, research papers, guidelines, and reports |
| Text extraction     | Extract, clean, normalize, and structure document text                           |
| Chunking strategies | Fixed-size, recursive, and semantic chunking                                     |
| Embeddings          | OpenAI embeddings and local sentence-transformer embeddings                      |
| Vector storage      | Qdrant for dense vector search with metadata filtering                           |
| Sparse retrieval    | BM25 keyword-based search                                                        |
| Hybrid retrieval    | Dense + sparse retrieval with reciprocal rank fusion                             |
| Query enhancement   | Query expansion, HyDE, and multi-query retrieval                                 |
| Reranking           | Cross-encoder reranking for top retrieved candidates                             |
| Answer generation   | LLM-generated responses grounded in retrieved evidence                           |
| Citations           | Document name, page number, chunk ID, and retrieval score                        |
| Evaluation          | RAGAS metrics, retrieval metrics, latency, and cost tracking                     |
| API-first design    | FastAPI backend with Swagger/OpenAPI documentation                               |
| Developer setup     | Docker Compose, `.env.example`, tests, linting, and CI/CD                        |

---

## System Architecture

![MedEvidence-RAG Architecture](https://github.com/wasay530/MedEvidence-RAG/blob/9e1afc90f2082e4fb321c675a37ecc80408a9383/docs/architecture/medevidence-rag-architecture.png)

---

## Architecture Layers

### 1. Data Sources & Input

Supported input formats:

* PDF research papers
* Clinical guidelines
* Thesis documents
* Technical reports
* Markdown files
* TXT files

Documents are uploaded through the FastAPI backend and processed asynchronously.

---

### 2. Application & Ingestion Layer

The ingestion layer handles document intake, preprocessing, and indexing preparation.

Main components:

* **FastAPI Backend**

  * REST API
  * Upload endpoints
  * Query endpoints
  * Evaluation endpoints
  * Swagger/OpenAPI documentation

* **Document Upload & Metadata Service**

  * Stores document-level metadata
  * Tracks file status
  * Assigns document IDs
  * Maintains source provenance

* **Celery Ingestion Worker**

  * Runs document processing asynchronously
  * Prevents upload requests from blocking
  * Supports scalable background processing

* **Text Extraction & Cleaning**

  * Extracts text from PDFs and plain documents
  * Removes malformed characters
  * Normalizes whitespace
  * Preserves page-level metadata where possible

---

### 3. Chunking Layer

The system supports multiple chunking strategies so retrieval performance can be benchmarked scientifically.

| Strategy           | Description                                                 | Use Case                        |
| ------------------ | ----------------------------------------------------------- | ------------------------------- |
| Fixed chunking     | Splits text into fixed token or character windows           | Simple baseline                 |
| Recursive chunking | Splits text using paragraph, sentence, and token boundaries | General-purpose RAG             |
| Semantic chunking  | Splits based on semantic similarity or topic shifts         | Research and clinical documents |

The goal is not only to chunk documents, but to measure how each chunking method affects retrieval quality.

---

### 4. Storage & Indexing Layer

MedEvidence-RAG uses separate storage systems for different responsibilities.

| Storage Component | Purpose                                                            |
| ----------------- | ------------------------------------------------------------------ |
| Qdrant            | Dense vector embeddings and metadata filtering                     |
| BM25 index        | Sparse keyword retrieval                                           |
| PostgreSQL        | Document metadata, query logs, evaluation runs, experiment results |
| Redis             | Celery queues, cache, temporary query/session state                |

---

### 5. Query & Retrieval Pipeline

The retrieval pipeline is the core of the system.

Query flow:

```text
User query
→ Query normalization
→ Optional query expansion
→ Optional HyDE generation
→ Optional multi-query generation
→ Dense vector retrieval
→ BM25 sparse retrieval
→ Hybrid fusion
→ Cross-encoder reranking
→ Top-k evidence selection
```

Supported retrieval modes:

| Mode                 | Description                                           |
| -------------------- | ----------------------------------------------------- |
| Dense only           | Vector similarity search using embeddings             |
| BM25 only            | Sparse keyword retrieval                              |
| Hybrid               | Dense + sparse search                                 |
| Hybrid + Rerank      | Hybrid retrieval followed by cross-encoder reranking  |
| HyDE + Hybrid        | Hypothetical document embedding before retrieval      |
| Multi-query + Hybrid | Multiple reformulated queries combined before ranking |

---

### 6. Generation Layer

The generation layer receives the top-k evidence chunks and creates a grounded answer.

The LLM is instructed to:

* Use only retrieved context
* Cite supporting chunks
* Avoid unsupported claims
* Return “insufficient evidence” when context is weak
* Provide confidence and latency metadata

Example response:

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
  "latency_ms": 1840
}
```

---

### 7. Evaluation & Benchmarking

The project includes an evaluation framework to compare retrieval and generation quality.

Evaluation dataset format:

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

Metrics:

| Metric Type | Metrics                                              |
| ----------- | ---------------------------------------------------- |
| Retrieval   | Recall@k, MRR, Context Precision, Context Recall     |
| Generation  | Faithfulness, Response Relevancy, Answer Correctness |
| System      | Latency, cost per query, token usage                 |

Planned benchmark format:

| Retrieval Strategy       | Context Precision | Context Recall | Faithfulness | Response Relevancy | Avg Latency |
| ------------------------ | ----------------: | -------------: | -----------: | -----------------: | ----------: |
| Dense only               |               TBD |            TBD |          TBD |                TBD |         TBD |
| BM25 only                |               TBD |            TBD |          TBD |                TBD |         TBD |
| Hybrid                   |               TBD |            TBD |          TBD |                TBD |         TBD |
| Hybrid + Reranker        |               TBD |            TBD |          TBD |                TBD |         TBD |
| HyDE + Hybrid + Reranker |               TBD |            TBD |          TBD |                TBD |         TBD |

Benchmark values should be added only after running the evaluation pipeline.

---

## Tech Stack

### Backend

* Python
* FastAPI
* Pydantic
* Uvicorn
* Celery
* Redis
* PostgreSQL
* SQLAlchemy

### Retrieval & LLM

* OpenAI API
* Sentence Transformers
* Qdrant
* BM25
* Cross-encoder reranker
* RAGAS
* Transformers
* scikit-learn

### Frontend & Developer Tools

* Streamlit demo UI
* Docker
* Docker Compose
* GitHub Actions
* Pytest
* Ruff
* Pre-commit
* Swagger/OpenAPI

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
│   │   ├── config.py
│   │   ├── logging.py
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
│   │   ├── bm25_retriever.py
│   │   ├── hybrid_retriever.py
│   │   ├── hyde.py
│   │   ├── multi_query.py
│   │   └── reranker.py
│   │
│   ├── generation/
│   │   ├── prompt_templates.py
│   │   └── answer_generator.py
│   │
│   ├── evaluation/
│   │   ├── ragas_eval.py
│   │   ├── benchmark_runner.py
│   │   └── report_writer.py
│   │
│   ├── workers/
│   │   └── celery_worker.py
│   │
│   └── db/
│       ├── models.py
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
│   └── test_api.py
│
├── data/
│   ├── sample_docs/
│   └── eval_sets/
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
  "answer": "Federated learning preserves privacy by keeping raw data on local clients and sharing model updates instead of centralizing sensitive data.",
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
  "latency_ms": 1840
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
```

### 3. Start services

```bash
docker compose up --build
```

### 4. Open the API docs

```text
http://localhost:8000/docs
```

### 5. Open the Streamlit demo

```text
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

| Service     | Purpose                          |
| ----------- | -------------------------------- |
| `api`       | FastAPI backend                  |
| `worker`    | Celery ingestion worker          |
| `redis`     | Queue and cache                  |
| `postgres`  | Metadata and experiment database |
| `qdrant`    | Vector database                  |
| `streamlit` | Demo UI                          |

---

## Environment Variables

| Variable             | Description                                 |
| -------------------- | ------------------------------------------- |
| `OPENAI_API_KEY`     | API key for OpenAI models                   |
| `DATABASE_URL`       | PostgreSQL connection URL                   |
| `REDIS_URL`          | Redis connection URL                        |
| `QDRANT_URL`         | Qdrant service URL                          |
| `EMBEDDING_PROVIDER` | `openai` or `sentence_transformers`         |
| `LLM_PROVIDER`       | LLM provider name                           |
| `DEFAULT_TOP_K`      | Number of chunks retrieved before reranking |
| `RERANK_TOP_K`       | Number of chunks kept after reranking       |
| `LOG_LEVEL`          | Logging level                               |

---

## Roadmap

### Phase 1 — MVP RAG Backend

* [ ] FastAPI application setup
* [ ] PDF upload endpoint
* [ ] Text extraction pipeline
* [ ] Fixed and recursive chunking
* [ ] OpenAI embeddings
* [ ] Qdrant vector indexing
* [ ] Basic citation-grounded query endpoint
* [ ] Docker Compose setup

### Phase 2 — Advanced Retrieval

* [ ] BM25 sparse retrieval
* [ ] Hybrid retrieval
* [ ] Reciprocal rank fusion
* [ ] Cross-encoder reranking
* [ ] Query expansion
* [ ] HyDE retrieval
* [ ] Multi-query retrieval

### Phase 3 — Evaluation

* [ ] Evaluation dataset
* [ ] RAGAS integration
* [ ] Retrieval metrics
* [ ] Generation metrics
* [ ] Benchmark report generation
* [ ] README benchmark table

### Phase 4 — Production Improvements

* [ ] Authentication
* [ ] Rate limiting
* [ ] Structured logging
* [ ] Query tracing
* [ ] Cost tracking
* [ ] CI/CD pipeline
* [ ] Deployment guide

---

## Example Use Cases

* Healthcare literature question answering
* Clinical guideline search
* Research paper summarization
* Thesis and technical report exploration
* Private document Q&A
* Evidence-grounded academic assistant
* RAG evaluation benchmarking

---

## What Makes This Project Different

MedEvidence-RAG is not only a chatbot over PDFs.

It is designed to show:

1. **Retrieval engineering** through dense, sparse, hybrid, and reranked search.
2. **Evaluation discipline** through RAGAS and retrieval metrics.
3. **Production mindset** through FastAPI, Docker, Celery, Redis, Qdrant, and PostgreSQL.
4. **Healthcare/research relevance** through evidence-grounded answers and citation-first design.
5. **Portfolio strength** through measurable benchmark results rather than vague claims.

---

## Suggested GitHub Topics

```text
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
