---
layout: page
permalink: /projects/earnings-call-intelligence-agent/
title: Earnings Call Intelligence Agent
---

## Earnings Call Intelligence Agent

- Built an AI-powered research backend for earnings call transcript analysis
- Ingests transcripts from local upload, Alpha Vantage, or Financial Modeling Prep
- Stores transcript chunks in **PostgreSQL + pgvector** for retrieval-augmented generation
- Answers analyst questions with citation-backed context and generates quarter-over-quarter change reports
- Uses **FastAPI**, **asyncpg**, **Pydantic**, **Docker Compose**, and OpenAI chat/embedding models
- [GitHub](https://github.com/Felix772/earnings-call-intelligence-agent)

> Research support only: this project does not provide investment advice, trading recommendations, or buy/sell/hold signals.

## Table of contents
- [Problem](#problem)
- [Architecture](#architecture)
- [Data Model](#data-model)
- [Core Workflows](#core-workflows)
- [API Surface](#api-surface)
- [Evaluation](#evaluation)
- [Testing](#testing)
- [Production Hardening](#production-hardening)

---

## Problem

Earnings calls contain useful qualitative signals, but manually comparing calls across quarters is slow and easy to bias. This project turns transcripts into a searchable research layer:

- ask questions against stored transcripts
- retrieve the exact transcript chunks used as evidence
- compare current and prior quarters
- surface changes in tone, demand, margins, guidance, costs, supply, competition, and risk

The main design constraint is groundedness: generated answers should stay tied to transcript evidence rather than becoming unsupported financial commentary.

---

## Architecture

The backend is organized as thin FastAPI routes over explicit service modules.

```text
Client
  |
  v
FastAPI
  GET  /health
  POST /ingest/local
  POST /ingest/alpha-vantage
  POST /ingest/fmp
  POST /ask
  POST /reports/change
  |
  +-- Ingestion services
  |     sources -> chunking -> embeddings -> storage
  |
  +-- RAG / reporting services
        retrieval -> LLM prompt -> validation -> persisted report

PostgreSQL + pgvector
  companies
  transcripts
  transcript_chunks
  reports
```

Key engineering choices:

- raw `asyncpg` SQL for direct control over pgvector syntax
- startup DDL for the MVP, with Alembic listed as the production migration path
- `text-embedding-3-small` embeddings stored as `vector(1536)`
- `gpt-4.1-mini` as the default chat model
- batched embedding calls to keep memory and provider usage bounded
- defensive Pydantic validation around LLM-generated JSON

---

## Data Model

The database keeps transcript content, embeddings, and generated reports separate:

| Table | Purpose |
|---|---|
| `companies` | Symbol registry and optional company names |
| `transcripts` | One row per company, fiscal year, and fiscal quarter |
| `transcript_chunks` | Chunked transcript text, token estimate, optional speaker/section, and vector embedding |
| `reports` | Persisted JSONB change reports |

The project creates a pgvector IVFFlat index on transcript embeddings:

```sql
CREATE INDEX IF NOT EXISTS idx_chunks_embedding
ON transcript_chunks
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 50);
```

For an MVP this is enough to make vector retrieval explicit and reproducible. For larger datasets, the README calls out index tuning and `VACUUM ANALYZE` as follow-up work.

---

## Core Workflows

### Ingestion

```text
raw transcript
  -> clean whitespace
  -> split into 500-word chunks with 60-word overlap
  -> estimate tokens
  -> embed in batches
  -> upsert company/transcript
  -> replace old chunks for that quarter
  -> bulk insert chunk rows
```

The chunking strategy preserves context across chunk boundaries while keeping each retrieved item small enough for prompt construction.

### Retrieval-Augmented Q&A

The `/ask` flow is:

```text
question
  -> OpenAI embedding
  -> pgvector cosine search filtered by symbol
  -> top-k transcript chunks
  -> context block with citation ids
  -> LLM answer with required citations
```

The retrieval query orders by pgvector distance and returns similarity, fiscal period, symbol, speaker, and content. The response exposes both the generated answer and the citation list so a reader can inspect the evidence.

### Change Reports

The `/reports/change` flow compares the requested quarter to the immediately prior quarter:

- Q1 compares against Q4 of the previous year
- Q2, Q3, and Q4 compare against the previous quarter in the same year
- up to 20 chunks are loaded from each transcript in chunk order
- current evidence is labeled `CUR1`, `CUR2`, etc.
- prior evidence is labeled `PRI1`, `PRI2`, etc.
- the LLM must return a strict JSON report
- Pydantic validates and normalizes the report before persistence

Report fields include:

- executive summary
- change score from 1 to 10
- tone shift direction and explanation
- key changes by theme
- risk flags with severity
- follow-up questions for an analyst

---

## API Surface

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/health` | Service status |
| `POST` | `/ingest/local` | Ingest raw transcript text |
| `POST` | `/ingest/alpha-vantage` | Fetch and ingest a provider transcript |
| `POST` | `/ingest/fmp` | Fetch and ingest a provider transcript |
| `POST` | `/ask` | Answer a question with cited transcript chunks |
| `POST` | `/reports/change` | Generate a structured quarter-over-quarter report |

Local setup runs through Docker Compose:

```bash
docker compose up --build
```

The API serves interactive docs at `http://localhost:8000/docs`.

---

## Evaluation

The evaluation guide treats this as a retrieval and grounded-generation system, not just a working API.

Recommended metrics and checks:

- **Retrieval hit rate**: compare retrieved chunks against manually selected gold passages
- **Citation quality**: ensure every `[C1]`-style reference exists in the citation list
- **Groundedness**: check that answer claims are supported by returned chunks
- **Change report quality**: verify tone, key changes, risk flags, and follow-up questions
- **Latency targets**: track p50 and p95 for health, ask, report, and ingest endpoints
- **Cost per report**: estimate embedding and LLM usage per generated report

The project also defines a manual checklist after loading synthetic `XYZ` data, including:

- `/health` returns ok
- Q3 and Q4 transcripts ingest successfully
- `/ask` returns answer text with citations
- `/reports/change` returns valid JSON
- no output includes buy/sell/hold recommendations

---

## Testing

The unit tests avoid OpenAI and database calls, which keeps the core logic testable in a restricted local environment.

Covered areas:

- health endpoint
- chunking behavior
- empty and whitespace-only transcript handling
- chunk index sequencing
- overlap between adjacent chunks
- schema validation for ingest, ask, and report requests
- ticker normalization
- fiscal-quarter bounds
- prior-quarter wraparound logic

Run tests from the backend:

```bash
cd backend
pip install -r requirements.txt
pytest tests/ -v
```

---

## Production Hardening

The README tracks the main production work still needed:

- authentication with API keys or JWT middleware
- rate limiting around ingestion and question-answering routes
- async task queue for long transcript ingestion
- pgvector IVFFlat tuning as data grows
- transcript provider caching
- structured logging, tracing, and metrics
- per-request token budgets and usage tracking
- Alembic migrations instead of startup DDL
- managed secrets instead of `.env`
- provider retry logic with exponential backoff

This makes the project useful as a backend/RAG prototype while still being honest about the gap between a working MVP and a production research platform.
