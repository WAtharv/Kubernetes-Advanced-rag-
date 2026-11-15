# Kubernetes SRE Copilot (Advanced RAG on LangGraph)

[![Backend](https://img.shields.io/badge/Backend-Stage%205%20Complete-brightgreen.svg)]()
[![Frontend](https://img.shields.io/badge/Frontend-Planned%20%2F%20Under%20Development-orange.svg)]()
[![Python](https://img.shields.io/badge/Python-3.11%2B-blue.svg)]()
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115%2B-009688.svg)]()
[![LangGraph](https://img.shields.io/badge/LangGraph-0.2.60%2B-blueviolet.svg)]()
[![Qdrant](https://img.shields.io/badge/Qdrant-Embedded%20%2F%20Vector-dc2626.svg)]()
[![SQLite](https://img.shields.io/badge/SQLite-Text2SQL%20Store-003B57.svg)]()

> **Current Project Status:**
> - **Backend:** Stage 5 complete (Advanced RAG, Hybrid Retrieval, Reranking, CRAG/Self-RAG, Secure Text2SQL with Human Approval & Audit Trail)
> - **Frontend:** Planned / under development (original baseline UI served as static assets by FastAPI at `/`)

---

## 1. Project Overview & Use Case

The **Kubernetes SRE Copilot** is an enterprise-grade AI operations assistant engineered for Site Reliability Engineers (SREs) and platform teams managing containerized Kubernetes infrastructure. 

When incidents occur (e.g. `CrashLoopBackOff`, `OOMKilled`, node readiness degradation, ingress routing failures), engineers must rapidly synthesize unstructured runbooks, release documentation, architecture guides, and structured cluster state. 

The Copilot provides a unified, deterministic, and safe natural language interface with two complementary paths:
1. **Unstructured Knowledge Path (Advanced RAG):** Ingests, indexes, and retrieves technical documentation and runbooks across Kubernetes, Helm, Prometheus, and internal postmortems.
2. **Structured Operational Path (Secure Text2SQL):** Converts natural-language questions about clusters, nodes, deployments, and SLO breaches into safe SQL queries, validates AST safety, enforces read-only transactions, requires explicit human approval before execution, and records every action in a tamper-evident audit log.

![Project Workflow](./Project Workflow.png)

---

## 2. Core Capabilities Completed (Stage 0 – Stage 5)

### Advanced RAG Pipeline (Stage 1 – Stage 4)
- **Multi-Format Ingestion:** Production parsers for Markdown, PDF, DOCX, HTML, CSV, JSON, and raw TXT with structural heading preservation and sliding windows.
- **Corpus Coverage:** 252 curated SRE documents spanning Kubernetes 1.31 architecture, Helm chart operations, Prometheus PromQL alerting, and simulated incident postmortems.
- **Hybrid Retrieval:** Dense 384-dimensional embeddings (`BAAI/bge-small-en-v1.5` via CPU-accelerated FastEmbed) fused with pure-Python BM25 sparse lexical search using Reciprocal Rank Fusion (RRF) or client-side weighted fusion.
- **Cross-Encoder Reranking:** High-precision neural reranking via `Xenova/ms-marco-MiniLM-L-6-v2` with calibrated sigmoid probability bounds.
- **HyDE (Hypothetical Document Embeddings):** Generates hypothetical runbook passages to bridge vocabulary gaps for conceptual troubleshooting queries.
- **Corrective RAG (CRAG):** Calibrated confidence scoring with autonomous fallback to query transformation and web search when local context confidence is low.
- **Self-RAG (Self-Reflective RAG):** Adaptive self-evaluation nodes that critique retrieval relevance, factual grounding, and citation coverage prior to returning final answers.
- **Deterministic Offline Fallback:** Fully functional no-LLM mode extracting verbatim runbook citations when network or API providers are unavailable.

### Secure Text2SQL & Human-in-the-Loop Workflow (Stage 5)
- **Dynamic Schema Introspection:** Automatic extraction of operational tables (`clusters`, `nodes`, `deployments`, `incidents`, `slo_breaches`).
- **AST Security Validation (SQLGlot):** Strict parsing rejecting DDL, DML (`UPDATE`, `INSERT`, `DELETE`, `DROP`, `ALTER`), CTE mutations, multi-statement injection, and unapproved SQL functions.
- **Read-Only Transaction Isolation:** Execution against SQLite with `mode=ro&uri=true`, query timeout bounding, and explicit rollback guarantees.
- **Human Approval Gate:** State machine interruption (`awaiting_approval`) preventing SQL execution until an authorized operator issues a signed approval or rejection.
- **Audit Logging:** Every query proposal, validation outcome, approver identity, execution latency, and row count is persisted in an audit log.

### 9-Layer Security Guardrail Pipeline
1. **Shape Inspection:** Validates query structure and bounds payload size.
2. **PII Redaction:** Strips sensitive credentials, IP addresses, and personal identities.
3. **Secret Protection:** Detects and blocks attempts to extract tokens, private keys, or passwords.
4. **Prompt Injection Defense:** Neutralizes system-prompt overrides and jailbreak syntax.
5. **Intent Classification:** Verifies SRE and operational intent.
6. **Scope Validation:** Ensures queries pertain to authorized infrastructure topics.
7. **Destructive Command Mitigation:** Flags risky remediation commands with change-control notices.
8. **Egress Filtering:** Scrubbing sensitive cluster data before client delivery.
9. **Grounding & Safety Review:** Post-generation hallucination and safety validation.

---

## 3. System Architecture

```text
                               +-----------------------------+
                               |     SRE User / Engineer     |
                               +-----------------------------+
                                              |
                                     HTTP / REST API (8000)
                                              v
+-----------------------------------------------------------------------------------------+
|                               FastAPI Application Gateway                               |
|                                                                                         |
|   GET /health    POST /ask    POST /retrieve    POST /approve    Static Web UI (/)      |
+-----------------------------------------------------------------------------------------+
                                              |
                                              v
+-----------------------------------------------------------------------------------------+
|                               LangGraph State Orchestrator                              |
|                                                                                         |
|  [Input Guardrails] -> [Router Node]                                                    |
|                              |                                                          |
|              +---------------+---------------+                                          |
|              |                               |                                          |
|              v (RAG Path)                    v (Text2SQL Path)                          |
|     [HyDE Query Expander]           [Text2SQL Generator (Groq)]                         |
|              |                               |                                          |
|     [Hybrid Retriever]              [SQLGlot AST Validator]                             |
|     (Dense 384d + BM25 RRF)                  |                                          |
|              |                      [Human Approval Gate]                               |
|     [Cross-Encoder Rerank]          (awaiting_approval interrupt)                       |
|              |                               |                                          |
|     [CRAG Quality Grader]           [Read-Only SQL Executor]                            |
|              |                               |                                          |
|     [Synthesis & Citations]         [Audit Trail Recorder]                              |
|              |                               |                                          |
|              +---------------+---------------+                                          |
|                              |                                                          |
|                              v                                                          |
|                 [Output Guardrails & Review]                                            |
+-----------------------------------------------------------------------------------------+
           |                                                      |
           v                                                      v
+-----------------------+                              +-----------------------+
|  Embedded Qdrant Store|                              |   SQLite Database     |
|   (data/qdrant/)      |                              |    (data/ops.db)      |
|  - sre_copilot_v1     |                              |  - clusters           |
|  - k8s_ops            |                              |  - nodes              |
|  - k8s_ops_bge_small  |                              |  - deployments        |
+-----------------------+                              |  - incidents          |
                                                       |  - slo_breaches       |
                                                       +-----------------------+
```

---

## 4. Repository Structure

The repository is organized into a clean `backend/` and `frontend/` separation:

```text
advanced-rag/
├── backend/                               # Complete Python / FastAPI backend application
│   ├── alembic/                           # Database schema migrations
│   │   ├── versions/
│   │   │   ├── 001_baseline_registry.py
│   │   │   └── 002_approval_audit_table.py
│   │   └── env.py
│   ├── data/                              # Knowledge corpus and operational databases
│   │   ├── corpus/                        # 252 curated Markdown runbooks & documentation
│   │   ├── legacy_corpus_backup/          # 9 baseline documents (82 chunks)
│   │   └── ops.db                         # SQLite operational database (excluded from git)
│   ├── docs/                              # Technical reports, benchmarks & validation logs
│   ├── eval_results/                      # Stage 4 retrieval benchmark datasets & figures
│   ├── scripts/                           # Database seeding, ingestion & benchmarking runners
│   │   ├── ingest_sre_copilot.py          # Complete 252-document corpus ingestion runner
│   │   ├── evaluate_sre_copilot.py        # Stage 4 benchmark evaluation runner
│   │   └── verify_storage.py              # Storage and collection health check utility
│   ├── src/advanced_rag/                  # Core application package
│   │   ├── api/                           # FastAPI endpoints, request models & static routing
│   │   ├── db/                            # SQLAlchemy models, sessions & engine management
│   │   ├── evaluation/                    # Retrieval metrics, MRR, NDCG@5, Hit@k
│   │   ├── graph/                         # LangGraph state machine, nodes & conditional edges
│   │   ├── guardrails/                    # 9-layer security pipeline
│   │   ├── ingestion/                     # Multi-format parsers, chunkers & corpus sync
│   │   ├── llm/                           # Groq, Ollama, and extractive fallback clients
│   │   ├── retrieval/                     # HybridStore, Dense BGE, local BM25, CrossEncoder
│   │   ├── text2sql/                      # Generator, SQLGlot AST validator, executor & audit
│   │   ├── config.py                      # Pydantic Settings & environment resolution
│   │   └── models.py                      # Core domain data models
│   ├── tests/                             # Comprehensive test suite (unit, integration & blackbox)
│   ├── .env.example                       # Safe environment variable template
│   ├── alembic.ini                        # Alembic migration configuration
│   ├── pyproject.toml                     # Package metadata, tools & pytest configuration
│   └── requirements.txt                   # Production and testing dependencies
├── frontend/                              # Original baseline web application
│   ├── index.html                         # USWDS-inspired single page interface
│   ├── app.js                             # Client logic, chat state, approval UI & lab
│   ├── markdown.js                        # Pure JavaScript markdown parser
│   └── styles.css                         # Application styling, dark mode & components
├── assets/                                # UI screenshots and architectural diagrams
├── Project Workflow.png                   # System architecture diagram
├── README.md                              # Repository documentation
└── .gitignore                             # Git ignore rules for secrets, DBs & caches
```

---

## 5. Technology Stack

- **Runtime & Language:** Python 3.11+
- **Application Framework:** FastAPI 0.115+, Uvicorn 0.32+
- **Graph Orchestration:** LangGraph 0.2.60+
- **LLM Provider (Default):** [Groq Cloud](https://console.groq.com) (`llama-3.3-70b-versatile`, `llama-3.1-8b-instant`)
- **Vector Storage:** Qdrant Client 1.12+ (Embedded on local filesystem)
- **Dense Embeddings:** FastEmbed (`BAAI/bge-small-en-v1.5`, 384 dimensions)
- **Sparse Embeddings:** Pure-Python BM25 with `snowballstemmer` Porter2 English stemmer
- **Reranker:** FastEmbed Cross-Encoder (`Xenova/ms-marco-MiniLM-L-6-v2`)
- **Text2SQL Engine:** SQLite 3 (local `data/ops.db`), SQLAlchemy 2.0+
- **SQL Security & AST Parsing:** SQLGlot 30.0+
- **Testing & Quality:** Pytest 8.3+, Pytest-Asyncio, Ruff, Black

---

## 6. Windows Setup & Run Instructions

### Step 1: Prerequisites
- Windows 10/11 with PowerShell
- Python 3.11+ installed and available on `PATH`
- A free [Groq API Key](https://console.groq.com)

### Step 2: Virtual Environment Setup
Open PowerShell in the repository root (`advanced-rag`):

```powershell
# Create virtual environment (if not already created)
python -m venv .venv

# Activate virtual environment
.\.venv\Scripts\Activate.ps1
```

### Step 3: Install Dependencies
Install requirements and register the backend in editable mode:

```powershell
# Install declared Stage 5 requirements
pip install -r backend/requirements.txt

# Install backend package in editable mode
pip install -e backend
```

### Step 4: Configure Environment Variables
Copy `.env.example` to `.env` in both the repository root and `backend/`:

```powershell
cp backend/.env.example backend/.env
cp backend/.env.example .env
```

Open `.env` and set your Groq API key:
```dotenv
GROQ_API_KEY=gsk_your_groq_api_key_here
LLM_PROVIDER=groq
GROQ_MODEL=llama-3.3-70b-versatile
```

### Step 5: Initialize & Seed the SQLite Database
Seed the operational database with sample clusters, nodes, deployments, and incidents:

```powershell
python -c "from advanced_rag.text2sql import seed_database; seed_database()"
# Output: Seeded operations database (SQLite): 416 rows in backend/data/ops.db
```

### Step 6: Ingest Knowledge Base into Embedded Qdrant
Ingest the curated SRE corpus into local embedded Qdrant:

```powershell
# Fast baseline ingestion (82 chunks into k8s_ops & k8s_ops_bge_small)
python -m advanced_rag.ingestion.cli data/legacy_corpus_backup --collection k8s_ops_bge_small

# Full 252-document corpus ingestion into sre_copilot_v1
python backend/scripts/ingest_sre_copilot.py
```

---

## 7. Everyday Development Commands

### Starting the Backend Application
Start the FastAPI application with Uvicorn:

```powershell
python -m uvicorn advanced_rag.api.main:app --host 127.0.0.1 --port 8000 --reload
```

### Accessing the Interfaces
- **Web UI:** Open [http://127.0.0.1:8000](http://127.0.0.1:8000) in your browser.
- **Interactive OpenAPI Documentation:** Open [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs).
- **Service Health Check:** Open [http://127.0.0.1:8000/health](http://127.0.0.1:8000/health).

### Running Tests
Execute the Stage 5 regression suite:

```powershell
# Run all Stage 5 tests
python -m pytest backend/tests/ -q --tb=short

# Run specific functional areas
python -m pytest backend/tests/test_text2sql.py -v
python -m pytest backend/tests/test_stage5_tasks60_61.py -v
python -m pytest backend/tests/test_stage5_tasks62_63.py -v
python -m pytest backend/tests/test_sre_copilot_retrieval.py -v
```

---

## 8. Common Troubleshooting & Notes

- **Embedded Qdrant File Locks (`EmbeddedStoreBusyError`):**  
  Embedded Qdrant creates an exclusive process lock file in `backend/data/qdrant/.lock`. Only one process may access the embedded store at a time. Do not run `ingest_sre_copilot.py` and `uvicorn` concurrently. If a crash leaves a stale lock, ensure no `python.exe` process is running before restarting.
- **Database Location:**  
  The SQLite Text2SQL database resolves dynamically to `backend/data/ops.db`. It is ignored by Git to prevent committing machine-specific state.
- **Corporate TLS Certificates:**  
  The application automatically initializes `truststore` at startup, ensuring seamless operation behind enterprise TLS inspection proxies without disabling SSL verification.

---

## 9. License

Proprietary — Internal Operations Tool.
