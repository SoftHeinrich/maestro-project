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
└── ArchitectureIssueRetrieval/  # Vector search research (FAISS + OpenAI embeddings)
```

Each subdirectory is its own git repository. They share a Docker network (`maestro_traefik`) and communicate via REST APIs through Traefik.

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

### ArchitectureIssueRetrieval
```bash
cd ArchitectureIssueRetrieval
pip install -r requirements.txt
python -m app.main ingest --data-dir ./data/issues_texts --store-dir ./store
python -m app.main query --store-dir ./store/token "your query"
python -m eval.evaluate --store-dir store --data-dir data/issues_texts
```

## Architecture

### Service communication
All services sit behind a Traefik reverse proxy (port 4269) with TLS. Path-based routing strips prefixes before forwarding:
- `/archui/` → Web UI (5173)
- `/issues-db-api/` → FastAPI backend (8000)
- `/search-engine/` → Search availability proxy (8042)
- `/pylucene/` → PyLucene search (8043)
- `/dl-manager/` → Deep learning service (9011)

### Issues DB API (`maestro-issues-db/issues-db-api/`)
FastAPI application with modular routers under `app/routers/`. JWT authentication. Uses MongoDB (issue data, predictions, ML models via GridFS) and PostgreSQL (relational data). Tests co-located with routers as `test_*.py` files using FastAPI TestClient.

### Deep Learning Manager (`maestro-dl-manager/dl_manager/`)
Largest codebase (~82 Python files). Key abstractions:
- **Feature generators**: Pluggable text vectorizers (Word2Vec, Doc2Vec, TF-IDF, BOW, ontology-based) registered via factory pattern
- **Classifiers**: BERT, RNN, CNN, fully connected networks (TensorFlow/Keras)
- **Accelerator**: Rust module (PyO3 + rayon) for parallel text preprocessing, built via `setuptools-rust`
- **Config system**: Complex validated schemas for model configurations with hyperparameter optimization (Keras Tuner)

Runs as a FastAPI web server or CLI tool (`python -m dl_manager`).

### Search Engine (`maestro-search-engine/`)
PyLucene-based full-text search. The availability proxy (`search-proxy/`) provides mutual exclusion for concurrent index/search requests. Supports prediction-filtered search using DL model outputs.

### Web UI (`maestro-ArchUI/src/`)
React 18 + Vite + Tailwind CSS. Route-based structure under `src/routes/`, reusable components in `src/components/`. Connection settings stored in localStorage, defaults in `src/components/connectionSettings.tsx`. Served under base path `/archui/` (configured in `vite.config.js`).

### ArchitectureIssueRetrieval (`ArchitectureIssueRetrieval/app/`)
Strategy pattern with `ChunkingStrategy` ABC (token/sentence/issue-aware chunking). FAISS IndexFlatIP vector stores with L2-normalized OpenAI embeddings. See its own `CLAUDE.md` for details.

## Known Constraints

- **passlib/bcrypt**: Pin `bcrypt==4.1.3` in issues-db-api (passlib 1.7.4 crashes with bcrypt 5.x)
- **Rust version**: DL Manager requires exactly Rust 1.85.0 (older can't parse edition 2024; 1.93+ breaks `issue_db_api` compilation)
- **PostgreSQL**: Pin to version 16 (18+ changed data directory format)
- **Base images**: Use `*-bookworm` not `*-buster` (Debian Buster is EOL)
- **Config secret**: `maestro-issues-db/issues-db-api/app/config.py` must contain `SECRET_KEY` (generated via `openssl rand -hex 32`)
- **Self-signed TLS**: DL Manager needs `DL_MANAGER_ALLOW_SELF_SIGNED_CERTIFICATE=TRUE` env var for local deployment
