# AI-First CRM HCP

![React](https://img.shields.io/badge/React-19-149ECA?logo=react&logoColor=white)
![FastAPI](https://img.shields.io/badge/API-FastAPI-009688?logo=fastapi&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![LangGraph](https://img.shields.io/badge/Agents-LangGraph-1C3C3C)

A full-stack CRM for pharmaceutical field representatives to record interactions with healthcare professionals (HCPs), review engagement activity, and use an AI assistant for CRM tasks and questions about local product documents.

The application combines a React dashboard, a FastAPI backend, SQLAlchemy models, and a LangGraph workflow. Groq handles language-model requests; Google GenAI supplies document and query embeddings for retrieval.

> The application identifies itself as **AI-First CRM HCP** in its source. The connected GitHub repository is named `DigiLocker-For-Hospitals`.

**Navigation:** [Features](#features) · [Technology stack](#technology-stack) · [Architecture](#architecture) · [Project structure](#project-structure) · [Getting started](#getting-started) · [Configuration](#environment-configuration) · [Usage](#usage-guide) · [API](#api-reference) · [Database](#database-overview) · [Checks](#scripts-and-validation) · [Limitations](#current-limitations) · [License](#license-and-credits)

## Overview

Field interactions contain useful information about doctors, products, meeting outcomes, and future visits. This project provides structured forms and natural-language tools to capture those records, retrieve previous discussions, and summarize activity in one interface.

The repository implements a local development application. Features below are verified in the source; the [current limitations](#current-limitations) explain integration gaps that affect runtime behavior.

## Features

| Capability | Implemented behavior |
| --- | --- |
| Interaction records | Create records with HCP details, interaction date/type, summary, product, outcome, follow-up date, and notes. Review, search, edit core fields, and delete records from history. |
| HCP search | Search doctor names from the dashboard. The chat search tool also searches hospital and specialization, with mappings such as “cardiologists” to “cardiology.” |
| AI CRM assistant | Classify messages and route them to tools for logging, editing the latest selected HCP interaction, search, next-best-action suggestions, follow-up updates, and recalling session context. |
| Document questions | Rewrite queries, combine FAISS semantic search with BM25 keyword search, rerank results with Groq, and generate answers from retrieved PDF context. The API returns source paths and retrieval scores. |
| Dashboard analytics | Display HCP and interaction counts, product distribution, follow-up KPIs, monthly trends, top doctors, a product leaderboard, and a weekday heatmap. |
| Refresh and exports | Poll dashboard statistics/trends every 30 seconds and listen for WebSocket update events. Export the dashboard's latest five matching interactions to CSV and a dashboard report to PDF. |
| Themes | Switch between light and dark themes; store the preference in browser local storage. |

Follow-up scheduling updates a date on an existing interaction. There is no background reminder delivery or calendar integration.

## Technology stack

| Layer | Technologies used by the implementation |
| --- | --- |
| Frontend | JavaScript, React 19, Vite 8, React Router, Redux Toolkit, React Redux |
| Interface and forms | Material UI, Emotion, MUI X Charts, React Hook Form, React Highlight Words |
| HTTP and exports | Axios, jsPDF, jsPDF AutoTable, browser Blob/download APIs |
| Backend | Python, FastAPI, Uvicorn, Pydantic, python-dotenv |
| Persistence | SQLAlchemy 2, PostgreSQL through `psycopg2-binary` |
| AI orchestration | LangGraph, Groq SDK, prompt templates and Python tools |
| Retrieval | LangChain PDF loaders/text splitters, Google GenAI (`gemini-embedding-001`), NumPy, FAISS, BM25, PyPDF |
| Cache and live updates | Redis for dashboard statistics; FastAPI WebSockets for dashboard events |
| Development checks | ESLint; pytest and FastAPI TestClient |

Dependency manifests are [frontend/package.json](frontend/package.json) and [backend/requirements.txt](backend/requirements.txt). The frontend has an npm lockfile. Several additional AI libraries are declared in the backend requirements; the active embedding code uses Google GenAI.

## Architecture

```mermaid
flowchart TD
    UI[React application] -->|Axios REST requests| API[FastAPI routers]
    UI <-->|Dashboard WebSocket| WS[Connection manager]
    API -->|Interaction route events| WS
    API --> ORM[SQLAlchemy]
    ORM --> DB[(PostgreSQL)]
    API <-->|Dashboard statistics cache| CACHE[(Redis)]
    API -->|Chat requests| GRAPH[LangGraph intent router]
    GRAPH <--> MEMORY[In-process session memory]
    GRAPH --> TOOLS[CRM tools]
    TOOLS --> ORM
    GRAPH --> RAG[RAG pipeline]
    GRAPH -->|Intent classification| GROQ[Groq]
    TOOLS --> GROQ
    RAG --> GROQ
    RAG --> GOOGLE[Google GenAI embeddings]
    RAG --> SEARCH[FAISS and BM25]
    PDF[Local PDF documents] --> SEARCH
```

- The frontend exposes `/`, `/log-interaction`, and `/history`. Chat is embedded in the Log Interaction page.
- `backend/app.py` mounts the API routers and initializes retrieval at startup. Importing `models.py` creates missing database tables.
- `backend/graph.py` is the active chat workflow. It routes detected intents to functions in `backend/tools/`; the search branch formats its result through a response-building node.
- The RAG pipeline loads local PDFs, splits them into 800-character chunks with 150-character overlap, and retrieves/reranks context before generating an answer. The existing FAISS index is loaded on startup; BM25 is built from the PDFs on each startup.
- Redis caches `/dashboard/stats` results for 60 seconds. The Redis wrapper skips cache operations if it cannot connect.
- Session memory and WebSocket connections belong to one backend process. Session memory is separate from Redis and disappears on restart.

## Project structure

```text
.
├── README.md
├── backend/
│   ├── app.py                  # FastAPI entry point and system endpoints
│   ├── config.py               # Environment loading
│   ├── database.py             # SQLAlchemy engine and sessions
│   ├── models.py               # HCP and interaction tables
│   ├── schemas.py              # Active API schemas
│   ├── graph.py                # Active LangGraph workflow
│   ├── routes/                 # Chat, interactions, edits, HCPs, dashboard
│   ├── tools/                  # CRM and RAG tool functions
│   ├── services/               # Groq, intent classification, cache, memory
│   ├── rag/
│   │   ├── documents/          # Five bundled PDF documents
│   │   ├── faiss_index/        # Saved index and document chunks
│   │   └── *.py                # Loading, embedding, retrieval, generation
│   ├── prompts/                # Model instructions and extraction prompts
│   ├── agents/                 # Supporting and older agent code
│   ├── nodes/                  # Response builder
│   ├── middleware/             # Request logging
│   ├── utils/                  # Logging and exception handlers
│   ├── tests/                  # Configured pytest suite
│   ├── test_*.py               # Standalone exploratory scripts
│   ├── seed_hcp.py             # Sample HCP seed data
│   ├── recreate_db.py          # Destructive table reset utility
│   ├── redis/                  # Bundled Windows Redis executables/docs
│   ├── pytest.ini
│   └── requirements.txt
└── frontend/
    ├── src/
    │   ├── main.jsx            # Router, Redux, and theme providers
    │   ├── api/                # Axios client
    │   ├── routes/             # Application routes
    │   ├── pages/              # Dashboard, logging, and fallback pages
    │   ├── components/         # Forms, chat, history, charts, exports
    │   ├── redux/              # Interaction/chat state and async requests
    │   ├── hooks/              # Dashboard WebSocket connection
    │   ├── context/            # Theme provider
    │   ├── layouts/            # Shared page shell
    │   ├── theme/              # Layout and styling constants
    │   └── styles/             # Global styles
    ├── public/                 # SVG assets
    ├── package.json
    ├── package-lock.json
    ├── eslint.config.js
    └── vite.config.js
```

## Getting started

### Prerequisites

- **Python 3.12** with pip and virtual-environment support. `backend/.python-version` specifies `3.12.10`; the `/version` endpoint separately reports a hard-coded `3.13` string.
- **Node.js 24+** and npm meet the checked-in Vite and ESLint engine requirements.
- A running **PostgreSQL** server and an existing database accessible to the application.
- A **Groq API key** and a Groq model identifier available to that account.
- A **Google API key** with access to the embedding model used by `rag/embeddings.py`.
- **Redis** at `localhost:6379`, database `0`, to enable caching. The backend can continue without Redis caching.

### 1. Clone the repository

```bash
git clone https://github.com/Lokesh-1921/DigiLocker-For-Hospitals.git
cd DigiLocker-For-Hospitals
```

If you already have a checkout, start from its root directory.

### 2. Install backend dependencies

```bash
cd backend
python3.12 -m venv venv
source venv/bin/activate
python -m pip install -r requirements.txt
```

On Windows, use a Python 3.12 interpreter to create the environment and activate it with `venv\Scripts\Activate.ps1` in PowerShell. The requirements file is stored as UTF-16; preserve its encoding when editing it.

### 3. Configure services and credentials

Create `backend/.env` with the variables in [Environment configuration](#environment-configuration), replacing every placeholder. No `.env.example` is included. The root `.gitignore` excludes `.env`.

Create the PostgreSQL database referenced by `DATABASE_URL` before starting the application. The models create the `hcp` and `interaction` tables automatically; they do not create the database itself or migrate existing tables.

For optional caching, start your installed Redis server in a separate terminal:

```bash
redis-server
```

The executables under `backend/redis/` are Windows binaries; macOS and Linux require their own Redis installation.

### 4. Start the backend

Run from `backend/` with the virtual environment active:

```bash
uvicorn app:app --reload --host 127.0.0.1 --port 8000
```

Use this working directory because RAG document and index paths are relative to it.

| Local resource | URL |
| --- | --- |
| API information | <http://localhost:8000/> |
| Swagger UI | <http://localhost:8000/docs> |
| ReDoc | <http://localhost:8000/redoc> |
| OpenAPI schema | <http://localhost:8000/openapi.json> |

To add the three sample HCPs, run once against a fresh development database:

```bash
python seed_hcp.py
```

The seed script inserts Dr Sharma, Dr Mehta, and Dr Gupta on every run, so rerunning it creates duplicates. `recreate_db.py` drops all application tables and is not part of normal setup.

### 5. Start the frontend

In a separate terminal, from the repository root:

```bash
cd frontend
npm ci
npm run dev
```

Open the URL printed by Vite, normally <http://localhost:5173>. The HTTP client is configured for `http://localhost:8000`.

## Environment configuration

Example contents for **`backend/.env`**; these are placeholders, not credentials:

```dotenv
DATABASE_URL=postgresql+psycopg2://crm_user:replace_with_password@localhost:5432/crm_hcp
GROQ_API_KEY=replace_with_your_groq_key
GROQ_MODEL=replace_with_an_enabled_groq_model_id
GOOGLE_API_KEY=replace_with_your_google_api_key
```

| Variable | Purpose |
| --- | --- |
| `DATABASE_URL` | Required by `config.py` and `database.py` to create the SQLAlchemy engine. |
| `GROQ_API_KEY` | Required when the Groq service is instantiated during application import. |
| `GROQ_MODEL` | Model identifier sent to Groq for classification, extraction, recommendations, query rewriting, reranking, and generation; no default is configured. |
| `GOOGLE_API_KEY` | Required by the lazy Google GenAI client for embedding queries and rebuilding the document index. An existing index can load without calling the embedding API. |

The only frontend environment variable read by the current source is optional **`VITE_WS_URL`**. To override the dashboard socket, create `frontend/.env`:

```dotenv
VITE_WS_URL=ws://localhost:8000/ws/dashboard
```

The same socket URL is the fallback when this variable is absent. The REST base URL is hard-coded in [frontend/src/api/api.js](frontend/src/api/api.js); there is no implemented `VITE_API_URL` setting. Redis connection details are also hard-coded in [backend/services/redis_service.py](backend/services/redis_service.py).

## Usage guide

1. **Dashboard (`/`):** review counts and analytics, search HCPs by name, apply product/month filters, refresh results, and export reports. The CSV contains only the recent-interactions subset returned by the dashboard API.
2. **Log Interaction (`/log-interaction`):** fill in an HCP name, summary, and product, then add optional details and dates. Populate both date inputs in the current UI to avoid the empty-date issue described below.
3. **AI assistant:** use the chat panel on the same page. For example, search with `Find Dr Sharma`, log with `Met Dr Sharma today. Discussed CardioX.`, then ask `What should I do next?` or `Schedule follow up after two weeks.` Search or log an HCP first to establish context before asking the assistant to edit the latest interaction.
4. **Document questions:** try `What does the CardioX clinical study say?` The answer is generated from the local document collection, and the API includes retrieval metadata.
5. **History (`/history`):** search records by doctor, product, summary, or ID. Edit the HCP name, summary, product, or follow-up date; delete through the confirmation dialog.

The dashboard's monthly trend and weekday heatmap use **follow-up dates**, rather than interaction dates. Its “completed followups” count means dates earlier than today; there is no explicit completion-status field. Month grouping combines matching months across years. Product/month filters affect statistics and recent records; other analytics endpoints remain global.

### Document collection and indexing

The bundled PDFs are `CardioX_Brochure.pdf`, `CardioX_Clinical_Study.pdf`, `HeartCare_Clinical_Study.pdf`, `Hypertension_Treatment_Guidelines.pdf`, and `NeuroPlus_Product_Guide.pdf`, under [backend/rag/documents](backend/rag/documents).

Documents are managed as local files; no upload endpoint is implemented. Adding or changing PDFs does not automatically rebuild an existing FAISS index. To regenerate it after document changes, stop the backend and run from `backend/` with Google credentials configured:

```bash
python -c "from config import DATABASE_URL; from rag.vector_store import vector_store; vector_store.build(); vector_store.save()"
```

Importing `config` loads `.env` before the embedding client is used. This command replaces the saved index and chunks; restart the backend afterward so both retrieval indexes use the updated documents. Embedding and chat requests require network access and may incur provider usage charges.

## API reference

Use [Swagger UI](http://localhost:8000/docs) for request schemas and interactive examples after starting the backend.

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/` | API name, version, status, and documentation path |
| `GET` | `/health` | Static application health response |
| `GET` | `/version` | Hard-coded application/runtime metadata |
| `GET` | `/metrics` | HCP and interaction totals from the database |
| `POST` | `/chat/` | Process `{ "session_id": "...", "message": "..." }` through the graph |
| `POST` | `/interaction/` | Create a record; requires `session_id`, `hcp_name`, `summary`, and `product` |
| `GET` | `/interaction/` | List all records, newest ID first |
| `PUT` | `/interaction/{interaction_id}` | Requires the creation schema; updates HCP name, summary, product, and follow-up |
| `DELETE` | `/interaction/{interaction_id}` | Delete a record |
| `PUT` | `/edit/{interaction_id}` | Accept optional core fields; updates supplied non-null HCP name, summary, product, and follow-up |
| `GET` | `/hcp/search?doctor_name=Sharma` | Case-insensitive partial name search |
| `GET` | `/dashboard/stats` | Counts, product distribution, latest five records; optional `product`, `doctor`, and `month` query parameters |
| `GET` | `/dashboard/monthly` | Counts grouped by follow-up month |
| `GET` | `/dashboard/kpi` | Today's/past follow-ups, distinct products, average interactions per HCP |
| `GET` | `/dashboard/top-doctors` | Top ten HCP names by interaction count |
| `GET` | `/dashboard/product-leaderboard` | Top ten products by interaction count |
| `GET` | `/dashboard/doctor-heatmap` | Follow-up weekday counts per HCP name |
| WebSocket | `/ws/dashboard` | Connection messages, ping/pong heartbeats, and `dashboard_updated` events |

Most REST responses use `success`, `message`, `data`, and `error`. Check `success` as well as the HTTP status: several handlers return failure envelopes with their normal status code. The custom request-validation handler returns **400**. Successful interaction creation returns **201**.

Example creation request:

```bash
curl -X POST http://localhost:8000/interaction/ \
  -H 'Content-Type: application/json' \
  -d '{
    "session_id": "demo-session",
    "hcp_name": "Dr Sharma",
    "summary": "Discussed the CardioX clinical study.",
    "product": "CardioX",
    "interaction_date": "2026-10-13",
    "follow_up": "2026-10-27"
  }'
```

Optional dates should be omitted or sent as `null` when unused. Chat messages must contain 1–4,000 characters; session IDs must contain 1–100 characters. Chat responses include `intent`, `response`, `rewritten_query`, `sources`, and `scores`. Scores can be FAISS L2 distances, BM25 scores, or null; they are not a single normalized confidence metric.

## Database overview

The tables are defined in [backend/models.py](backend/models.py):

| Table | Columns |
| --- | --- |
| `hcp` | `id` (integer primary key), `name` (required string), `specialization`, `hospital` |
| `interaction` | `id` (integer primary key), `hcp_name` (required string), `specialization`, `hospital`, `interaction_date` (date), `interaction_type`, `summary` (text), `product`, `outcome`, `follow_up` (date), `notes` (text), `created_at` (datetime) |

SQLAlchemy relationships join `HCP.name` to `Interaction.hcp_name`. There is no database foreign-key constraint or unique-name constraint, and creating an interaction does not automatically insert an HCP. Consequently, HCP totals reflect the `hcp` table rather than distinct doctor names in interaction records.

Tables are created through `Base.metadata.create_all()` at model import. No migration framework is included. Redis stores temporary dashboard cache entries; FAISS and pickled chunks store the document retrieval index on disk.

## Scripts and validation

### Frontend scripts

Run from `frontend/` after `npm ci`:

| Command | Defined script |
| --- | --- |
| `npm run dev` | `vite` — development server |
| `npm run build` | `vite build` — production assets in `frontend/dist/` |
| `npm run lint` | `eslint .` — configured JavaScript/React checks |
| `npm run preview` | `vite preview` — locally preview built assets |

There is no frontend test script.

### Backend tests

pytest is not listed in `requirements.txt`. Install it in the backend environment, then run from `backend/`:

```bash
python -m pip install pytest
python -m pytest
```

`pytest.ini` selects `backend/tests/`. The suite covers system endpoints, metrics, interaction creation/listing, chat validation, and chat intents. These are integration tests: importing the application connects to the configured database, and the shared test setup initializes RAG. Tests can create/update records and call AI providers; use a dedicated development/test database and credentials.

The creation test currently omits the required `session_id`, so it does not match the active schema. Top-level `backend/test_*.py` files are exploratory scripts outside the configured suite, and some reference older symbols. No backend lint configuration is included.

### Validation snapshot

Checks performed during this documentation update on **13 October 2026**:

| Check | Result |
| --- | --- |
| Frontend dependency installation | `npm ci` succeeded after retrying a network connection reset. |
| Backend dependency installation | Requirements and pytest installed in an isolated Python 3.12 environment. |
| `npm run build` | Passed; Vite warned about a minified bundle larger than 500 kB. |
| `npm run lint` | Failed with 74 errors and 9 warnings in existing application source. |
| Backend pytest suite | Blocked during application import: `DATABASE_URL` was unset, so SQLAlchemy could not create its engine. No suite tests executed. |
| Documentation checks | Relative links, heading anchors, code fences, script/environment names, all 18 REST/WebSocket routes, and whitespace checks passed. |
| Source and example checks | All backend Python files parsed; the example creation payload validated; all five PDFs contained extractable text. |

These results do not establish end-to-end application behavior. No application source was changed for this documentation update.

## Build and hosting

The supported frontend build/preview workflow is:

```bash
cd frontend
npm run build
npm run preview
```

Vite produces static assets in `frontend/dist/`. No Dockerfile, deployment workflow, hosting-provider configuration, or production backend service configuration is included. Hosting beyond local development requires configuring the fixed REST address, WebSocket URL, backend services, and SPA route fallback. This repository does not establish a deployed service URL.

## Current limitations

| Area | Source behavior to account for |
| --- | --- |
| Doctor filter | The dashboard form sends `hcp`, while `/dashboard/stats` expects `doctor`; that UI filter does not currently filter the backend query. |
| Empty dates | The interaction form submits blank date inputs as empty strings, which the backend's date fields reject. Use populated dates in the form or omit/send null dates through the API. |
| Edit responses | `/edit/{id}` commits changes, then calls `asyncio.create_task()` inside a synchronous handler. This can fail without a running event loop and return an error after the database update. |
| Live refresh | REST interaction routes clear caches and publish dashboard events. Chat mutation tools do neither, so their changes may require refresh/cache expiry. The socket refresh callback updates statistics and the monthly trend, not every chart. |
| Sessions | The frontend starts with the shared `frontend-session` identifier. Backend memory is not persistent or isolated through authentication. |
| General chat | `CHAT` is a classifier label, but its graph branch ends without a response-generation node. Use the supported CRM/document intents. |
| Access control | No authentication or authorization flow is implemented; CORS currently allows all origins. |

## Suggested improvements

These are development suggestions, not implemented features:

- Align UI parameters, optional-date serialization, API error handling, and test payloads with the backend contracts.
- Consolidate mutation handling so chat and REST paths consistently invalidate caches and broadcast updates.
- Add persistent per-user session storage, authentication, and authorization.
- Introduce database migrations and stable HCP identifiers instead of name-based relationships.
- Add isolated tests with provider mocks, CI checks, and a documented deployment configuration.

## License and credits

No `LICENSE` file is included. FastAPI metadata names MIT, but that metadata alone does not supply repository license terms.

The frontend retains its original React/Vite template notes in [frontend/README.md](frontend/README.md). The implementation uses the open-source libraries listed above and the Groq and Google GenAI services. Contributor history is available through the repository's Git commits.
