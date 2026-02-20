# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Maestro is a microservices platform for finding and exploring Architectural Design Decisions (ADDs) in issue tracking systems. It combines full-text search, deep learning classification, and a web UI for browsing/annotating Jira issues. The `ArchitectureIssueRetrieval/` subproject is a separate vector search research tool.

## Repository Layout

```
maestro-project/
├── Maestro/                 # Traefik reverse proxy, TLS certs, orchestration, docs
├── maestro-issues-db/       # MongoDB + PostgreSQL databases, FastAPI REST API
├── maestro-search-engine/   # PyLucene full-text search + availability proxy
├── maestro-dl-manager/      # Deep learning training/prediction (Python + Rust)
├── maestro-ArchUI/          # React + Vite web frontend (Tailwind CSS)
└── archRag/                 # Vector search API + evaluation (FAISS + OpenAI embeddings)
```

Each subdirectory is its own git repository. They share a Docker network (`maestro_traefik`) and communicate via REST APIs through Traefik.

## Setup on a New Machine

```bash
# Clone with all submodules
git clone --recurse-submodules git@github.com:SoftHeinrich/maestro-project.git
cd maestro-project

# If already cloned without submodules:
git submodule update --init --recursive

# Create the shared Docker network
docker network create maestro_traefik

# Generate TLS certs (if not present)
cd Maestro/certs && openssl req -x509 -nodes -days 3650 \
  -newkey rsa:2048 -keyout maestro.localhost.key \
  -out maestro.localhost.crt -subj "/CN=maestro.localhost" && cd ../..

# Generate issues-db-api secret key
python3 -c "import secrets; print(f\"SECRET_KEY = '{secrets.token_hex(32)}'\")" \
  > maestro-issues-db/issues-db-api/app/config.py

# Set OpenAI API key for archRag (create .env in archRag/)
echo "OPENAI_API_KEY=sk-..." > archRag/.env

# Start all services
cd Maestro && ./setup_components.sh

# Restore MongoDB data (if you have the archives)
docker exec -i mongo mongorestore --gzip --archive < maestro-issues-db/mongodump-MiningDesignDecisions-lite.archive
docker exec -i mongo mongorestore --gzip --archive < maestro-issues-db/mongodump-JiraRepos_2023-03-07-16:00.archive

# Restore PostgreSQL comments database
gunzip -c maestro-issues-db/postgressCommentsdata.sql.gz | docker exec -i psql psql -U postgres -d issues

# Create required index for comment lookups (critical for search performance)
docker exec -i psql psql -U postgres -d issues -c \
  "CREATE INDEX IF NOT EXISTS idx_issues_comments_issue_id ON issues_comments(issue_id);"
```

## Commands

### Start all services
```bash
cd Maestro && ./setup_components.sh
```

### Start individual services
```bash
# Traefik reverse proxy
cd Maestro && docker compose up --build -d

# Databases + API
cd maestro-issues-db && docker compose up --build -d

# Search engine
cd maestro-search-engine && docker compose up --build -d

# Deep learning manager (no GPU / with GPU)
cd maestro-dl-manager && docker compose -f docker-compose-no-gpu.yml up --build -d
cd maestro-dl-manager && docker compose -f docker-compose-gpu.yml up --build -d

# Web UI
cd maestro-ArchUI && docker compose up --build -d
```

### Issues DB API tests
```bash
cd maestro-issues-db/issues-db-api
pytest app/routers/          # all tests
pytest app/routers/test_authentication.py  # single test file
```

### DL Manager direct run (requires Rust 1.85.0)
```bash
cd maestro-dl-manager
python setup.py build_ext --inplace
python -m dl_manager serve --keyfile server.key --certfile server.crt
```

### Frontend development
```bash
cd maestro-ArchUI
npm install && npm run dev       # local dev server
npm run build                    # production build
```

### archRag (vector search)
```bash
cd archRag
pip install -r requirements.txt && pip install -e .

# Ingest issues into FAISS stores (all 3 chunking strategies)
python -m app.main ingest --data-dir ./data/issues_texts --store-dir ./store

# Query a single strategy
python -m app.main query --store-dir ./store/issue "data replication"

# Run unit tests
pytest tests/ -v

# Run archRag API locally (port 8044)
python -m api
```

### Full evaluation (RAG vs PyLucene vs Original)

Compares 8 search systems using student experiment ground truth. Metrics: nDCG@k, P@k, MRR, First Hit Rank.

**Systems evaluated:**
- `original` — student experiment results (baseline)
- `pylucene` — raw query, no prediction filtering
- `pylucene_gpt` — GPT-4o-mini keyword extraction (5 keywords), no prediction filtering
- `pylucene_rerank` — raw query, with ADD type prediction reranking
- `pylucene_rerank_gpt` — GPT keywords + prediction reranking
- `rag-token`, `rag-sentence`, `rag-issue` — FAISS vector search with 3 chunking strategies

**Prerequisites:**
- FAISS stores built (`archRag/store/token/`, `sentence/`, `issue/`)
- PyLucene service running (`docker ps | grep pylucene`)
- OpenAI API key set in `archRag/.env` (for embeddings + GPT keyword extraction)

```bash
cd archRag

# Step 1: Cache query embeddings (one-time, calls OpenAI API)
python -m eval.embed_queries --model-name text-embedding-3-small

# Step 2a: Evaluate RAG strategies only (no live services needed)
python -m eval.evaluate --store-dir store --data-dir data/issues_texts

# Step 2b: Full comparison including live PyLucene API
# Phase 1: GPT keyword extraction (cached after first run)
# Phase 2: Pre-fetch PyLucene results (cached to disk, resumable)
# Phase 3: Evaluate all 8 systems
python -m eval.evaluate_all \
    --store-dir store \
    --data-dir data/issues_texts \
    --pylucene-url http://localhost:8043

# Options:
#   --threshold 3        Relevance threshold (rating >= 3 = relevant)
#   --exp-data PATH      Path to experiment JSON data
#   --model-name NAME    OpenAI embedding model (must match store)
#   --skip-pylucene      Evaluate only RAG systems
#   --fetch-only         Only pre-fetch PyLucene results, don't evaluate
```

**Output:**
```
System                   nDCG@5  nDCG@10      P@5     P@10      MRR      FHR    Unr@5   Unr@10
original                 0.7633   0.7409   0.7572   0.7281   0.8636     1.43     0.00     0.00
pylucene                 0.6866   0.6838   0.6791   0.6918   0.8137     1.40     2.12     4.72
pylucene_gpt             0.7649   0.7582   0.7549   0.7563   0.8865     1.36     1.99     4.36
pylucene_rerank          0.6636   0.6653   0.6614   0.6776   0.7966     1.48     1.99     4.61
pylucene_rerank_gpt      0.7376   0.7311   0.7341   0.7299   0.8553     1.45     1.87     4.29
rag-token                0.8557   0.8157   0.8496   0.8257   0.9368     1.16     3.01     6.43
rag-sentence             0.8597   0.8233   0.8543   0.8332   0.9434     1.15     2.99     6.33
rag-issue                0.8713   0.8397   0.8645   0.8455   0.9413     1.14     2.85     5.98
```

**Caching**: PyLucene results are cached to `eval/cache/pylucene_results.json`. If the process crashes, restart and it resumes from where it left off. GPT keywords are cached to `eval/cache/gpt_keywords.json`.

## Architecture

### Service communication
All services sit behind a Traefik reverse proxy (port 4269) with TLS. Path-based routing strips prefixes before forwarding:
- `/archui/` → Web UI (5173)
- `/issues-db-api/` → FastAPI backend (8000)
- `/search-engine/` → Search availability proxy (8042)
- `/pylucene/` → PyLucene search (8043)
- `/dl-manager/` → Deep learning service (9011)
- `/archrag/` → Vector search API (8044)

### Databases (`maestro-issues-db/`)

Two databases run side-by-side in `maestro-issues-db/docker-compose.yml`:

**MongoDB** (`mongo`, port 27017) — Issue metadata, ML models, predictions, tags, embeddings:
- `MiningDesignDecisions` DB: `IssueLabels`, `DLModels`, `Tags`, `Projects`, `DLEmbeddings`, `Files`, `RepoInfo` collections
- `JiraRepos` DB: One collection per ecosystem (e.g., `Apache`, `RedHat`) with raw Jira issue JSON
- Data dumps: `mongodump-MiningDesignDecisions-lite.archive`, `mongodump-JiraRepos_2023-03-07-16:00.archive`
- Admin UI: Mongo Express at `http://localhost:8081`

**PostgreSQL** (`psql`, port 5432) — Issue comments and comment-level classification results:
- Database: `issues`, user: `postgres`, password: `pass`
- Data volume: Named Docker volume `pgdata` (persists across container restarts)
- Data dump: `postgressCommentsdata.sql.gz` (~627 MB compressed)
- Admin UI: Adminer at `http://localhost:8082` (default server: `psql`)

PostgreSQL tables:
| Table | Columns | Description |
|-------|---------|-------------|
| `issues_comments` | `id`, `issue_id`, `author_name`, `author_display_name`, `body`, `is_bot` | ~4M issue comments from Jira |
| `classification_results` | `issue_comment_id`, `classification_result` | Comment-level ML predictions (existence/property/executive confidence scores) |

The `classification_result` column stores JSON: `{"existence": {"confidence": 0.8}, "executive": {"confidence": 0.3}, "property": {"confidence": 0.5}}`.

**Who uses PostgreSQL**: The PyLucene search engine (`maestro-search-engine/pylucene/app/adapter.py`) connects to PostgreSQL to:
1. Fetch comments during index building — concatenates comment text into Lucene documents for full-text search
2. Fetch comments + classification scores during search — used for reranking (Equation 3.1: weighted combination of text score, issue-level predictions, and comment-level predictions)

The query filters comments to `LENGTH(body) > 200 AND is_bot = false`, then LEFT JOINs `classification_results` for confidence scores.

**Connection config** (environment variables in `maestro-search-engine/docker-compose.yml`):
```
POSTGRES_HOST=psql    POSTGRES_PORT=5432
POSTGRES_DB=issues    POSTGRES_USER=postgres    POSTGRES_PASSWORD=pass
```

### Issues DB API (`maestro-issues-db/issues-db-api/`)
FastAPI application with modular routers under `app/routers/`. JWT authentication. Uses MongoDB (issue data, predictions, ML models via GridFS) and PostgreSQL (comments/classification via PyLucene). Tests co-located with routers as `test_*.py` files using FastAPI TestClient.

### Deep Learning Manager (`maestro-dl-manager/dl_manager/`)
Largest codebase (~82 Python files). Key abstractions:
- **Feature generators**: Pluggable text vectorizers (Word2Vec, Doc2Vec, TF-IDF, BOW, ontology-based) registered via factory pattern
- **Classifiers**: BERT, RNN, CNN, fully connected networks (TensorFlow/Keras)
- **Accelerator**: Rust module (PyO3 + rayon) for parallel text preprocessing, built via `setuptools-rust`
- **Config system**: Complex validated schemas for model configurations with hyperparameter optimization (Keras Tuner)

Runs as a FastAPI web server or CLI tool (`python -m dl_manager`).

### Search Engine (`maestro-search-engine/`)
PyLucene-based full-text search with concurrent read support via a `ReadWriteLock` (multiple searches run in parallel; only index builds are exclusive). Supports prediction-filtered search with reranking using DL model classification confidence scores. The availability proxy (`search-proxy/`) provides an additional layer for the web UI.

**PostgreSQL dependency**: Connects to the shared PostgreSQL database (see Databases section above) to fetch issue comments for index building and reranking. Uses `psycopg2` with connection parameters from environment variables.

**JVM threading**: PyLucene uses JCC (Java-C++ bridge). Call `lucene.initVM()` once per process, then `attachCurrentThread()` for additional threads. Never call `initVM()` per thread (causes SIGSEGV).

### Web UI (`maestro-ArchUI/src/`)
React 18 + Vite + Tailwind CSS. Route-based structure under `src/routes/`, reusable components in `src/components/`. Connection settings stored in localStorage, defaults in `src/components/connectionSettings.tsx`. Served under base path `/archui/` (configured in `vite.config.js`).

### archRag (`archRag/`)
- **app/**: Core library — chunking strategies (token/sentence/issue-aware), FAISS IndexFlatIP vector store, OpenAI embeddings, search aggregation
- **api/**: FastAPI REST API (port 8044, Traefik path `/archrag`), loads FAISS store at startup, enriches results with issue metadata and classification labels from issues-db-api
- **eval/**: Evaluation framework comparing 8 systems (3 RAG strategies, 4 PyLucene variants, original). Uses GPT-4o-mini for keyword extraction, disk caching for PyLucene results (resumable after crashes), and handles Lucene ParseException for special-character queries

## Known Constraints

- **passlib/bcrypt**: Pin `bcrypt==4.1.3` in issues-db-api (passlib 1.7.4 crashes with bcrypt 5.x)
- **Rust version**: DL Manager requires exactly Rust 1.85.0 (older can't parse edition 2024; 1.93+ breaks `issue_db_api` compilation)
- **PostgreSQL**: Pin to version 16 (18+ changed data directory format)
- **Base images**: Use `*-bookworm` not `*-buster` (Debian Buster is EOL)
- **Config secret**: `maestro-issues-db/issues-db-api/app/config.py` must contain `SECRET_KEY` (generated via `openssl rand -hex 32`)
- **Self-signed TLS**: DL Manager needs `DL_MANAGER_ALLOW_SELF_SIGNED_CERTIFICATE=TRUE` env var for local deployment
- **PostgreSQL index**: `issues_comments.issue_id` must be indexed for PyLucene reranking performance (`CREATE INDEX idx_issues_comments_issue_id ON issues_comments(issue_id)`)
- **PyLucene JVM**: Never call `lucene.initVM()` per thread — call once, then `attachCurrentThread()` for additional threads
