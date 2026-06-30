<p align="center">
  <img src="docs/assets/rag-codebase-logo.png" width="400">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Status-active-brightgreen">
  <img src="https://img.shields.io/badge/Source-private-lightgrey">
</p>

<p align="right">
  <img src="https://img.shields.io/badge/made_by-Alex_Almeida-0d1117?style=flat&logo=terminal&logoColor=58a6ff&labelColor=21262d">
</p>

# RAG Codebase

Web chat interface with RAG for codebase analysis, integrating semantic search ([cocoindex-code](https://github.com/cocoindex-io/cocoindex-code)) with Claude (Anthropic) via API.

> ⚠️ This is the **official repository** of the RAG Codebase project.  
> Beware of unofficial copies or modified versions that may contain unsafe files.

## 📌 Purpose

Enable Dev, QA, PO, and Support teams to ask natural language questions about source code, directly from the browser, without relying on IDEs or CLI tooling.

### Examples

- Dev:     *"Where is rule X validated?"*
- QA:      *"Which tables are affected by this flow?"*
- PO:      *"Which modules use this entity?"*
- Support: *"What does this warning/error mean?"*

## 🔑 Key Features

- Session-aware chat (per-tab history in the browser)
- Dynamic selection of repository, branch, and layers (back-end, front-end, and/or database)
- Side panel showing files/chunks used in each response
- Token-by-token response streaming
- Markdown rendering with syntax highlighting
- Dark mode
- Confirmation modal when switching context (repo/branch)
- Active context badge visible throughout the session
- Containerized with Docker for server deployment
- Question classifier (simple, complex, irrelevant) with automatic model selection (Sonnet or Opus)
- Hybrid search: semantic (cocoindex-code) + exact (Git) for file names, method names, class names, and string literals.
- Search, select, and add a file as context (one at a time)
- Automatic index warmup when selecting or switching contexts (repository/branch)

## 🧩 Architecture

```
Browser (HTML + React via CDN)
        │
        │  POST /warmup    (when selecting or switching repositories/branches)
        │  POST /chat      (query + session_id + repo + branch)
        │  GET  /stream    (SSE - token streaming)
        ▼
   FastAPI (Python)
        │
        ├─ /warmup: ccc daemon stop (previous project) → ccc search dummy
        │           (forces the index to be loaded into GPU memory before 1st query)
        │
        ├─ hybrid search per query:
        │   ├─ ccc search "<query>"        → semantic search (similarity-based)
        │   └─ exact search + ccc search --path → exact search for:
        │       ├─ git ls-files (file names with extensions)
        │       └─ git grep -wc (method names, class names, and string literals)
        │       ▼
        │  score-based merge (git grep: score proportional to the number of occurrences)
        │       ▼
        │  relevant chunks (file, excerpt, score)
        │
        ├─ builds prompt with chunks as context
        │
        └─ Anthropic API (Claude)
                ▼
        streaming response → SSE → browser
```

### Session Management

- A `session_id` is generated on page load and persisted per browser tab
- Message history is stored in server memory, keyed by `session_id`
- When switching repository or branch, a confirmation modal is displayed and the session history is discarded

### Hybrid Search

The search combines two complementary mechanisms to maximize the relevance of the chunks returned to Claude.

**Semantic search** (`ccc search`)
Finds code that is conceptually related to the query, even when it does not contain exact identifiers. This is ideal for natural language questions such as "where is the credit validation rule implemented?".

**Exact search** (`git grep + ccc search --path`)
Automatically triggered when the query contains any of the following patterns:
- **File names with supported extensions** recognized by cocoindex-code (e.g., `Empresa.java`, `EmpresaCadastroEditor.vue`)
- **Identifiers** matching common method or class naming conventions: CamelCase, snake_case, or `s/g/i` prefix followed by an uppercase letter (e.g., `sMakeIntegrarPedido`, `ContainerEmpresa`)
- **Double-quoted string literals** representing acronyms or fixed system messages (e.g., `"CFOP"`, `"Customer does not have sufficient credit limit"`)

> Exact-match chunks are ranked proportionally to the number of occurrences of the searched term within each file (Term Frequency). Files with more occurrences receive higher scores, ensuring that the most relevant file appears at the top of the merged results.

**Usage tips:**
- **File name with extension**: triggers `git ls-files` to locate the file by path and retrieve its chunks using the full query as the search input, maximizing the semantic relevance of the returned results: *"What are the attributes of `Empresa.java`?"* or *"Tell me about `EmpresaCadastroEditor.vue`."*
- **Exact method or class name**: triggers `git grep -wc` to locate files containing the identifier:
  *"Review the `sMakeIntegrarPedidoAprovadoNaoIntegrado` method."*
- **Double-quoted message**: triggers `git grep -c` to locate the string literal in the codebase:
  *"I'm getting the following error when trying to integrate an approved order: `"Customer does not have sufficient credit limit"`"*


### Context Warmup

When selecting or switching repositories/branches, RAGCodebase automatically performs a warmup cycle before enabling user input:

- 1. **Stop the daemon** (if `DAEMON_STOP_ON_CONTEXT_SWITCH=true`): releases the VRAM allocated to the previous project.
- 2. **Warmup search**: executes a dummy query (`__RAG_CODEBASE_SEEK_WARMUP_CCC_DAEMON__`) to force the daemon to load the new project's index into GPU memory.

> This eliminates the latency of the first real query, which would otherwise occur while the daemon loads the model during request processing.

## ⚙️ How It Works - Model Routing

Before calling the main Claude model, the backend traverses a decision tree to select the most appropriate model and avoid unnecessary API calls. The goal is to minimize token cost without compromising response quality.

### Decision Flow

![Decision Flow](https://raw.githubusercontent.com/alexfdealmeida/rag-codebase/main/docs/assets/rag-codebase-flow-light.svg)

### Route Summary Table

| Route | Condition | Semantic search (ccc search) | Question classification (Haiku) | Final model |
|---|---|:---:|:---:|---|
| File in context | `context_file` present | ✗ | ✗ | Opus |
| Complex question | match in `COMPLEX_TRIGGERS` | ✓ | ✗ | Opus |
| Question about the assistant | match in `ASSISTANT_TRIGGERS` | ✗ | ✗ | Sonnet |
| No relevant sources | 0 chunks above threshold | ✓ | ✗ | Sonnet |
| Irrelevant question | Haiku → `irrelevant` | ✓ | ✓ | Default response (no Claude) |
| Simple question | Haiku → `simple` | ✓ | ✓ | Sonnet |
| Complex question | Haiku → `complex` | ✓ | ✓ | Opus |

### Step-by-Step Details

- **File in context**
Triggered when the user selects a source in the side panel and clicks "Use as context". The full file content is injected into the prompt alongside the question. No new `ccc search` is executed (chunks from the previous search are reused as additional context). Opus is chosen because full-file analysis is the most demanding task.

- **`COMPLEX_TRIGGERS`**
A keyword list evaluated locally (no API cost). If the question matches any keyword, Opus is invoked directly and Haiku is skipped.

- **`ASSISTANT_TRIGGERS`**
A list of expressions indicating questions about the assistant itself. When detected, `ccc search` is skipped entirely (no relevant code to retrieve).

- **ccc search + threshold**
Executes semantic search against the local index. Chunks with a score below `MIN_SCORE_THRESHOLD` (0.35) are discarded. If no chunks remain, Sonnet responds directly without invoking Haiku.

- **Haiku (classifier)**
Invoked only when relevant sources exist and no keyword/trigger was matched. Classifies the question as `complex`, `simple`, or `irrelevant` using `max_tokens=10` and `temperature=0.0`. The cost is minimal (~300–400 input tokens).

- **`ROUTE_SKIP`**
When Haiku returns `irrelevant` (outside the code/system context), the default response is emitted directly via SSE, no call to the main Claude model is made and the chunk tokens are never sent.

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Backend | FastAPI (Python) |
| Frontend | React 18 via CDN (no build step) |
| Styling | Tailwind CSS via CDN |
| Markdown | marked.js via CDN |
| Syntax highlighting | highlight.js via CDN |
| Hybrid search | cocoindex-code + Git |
| LLM | Anthropic API - Claude (Haiku, Sonnet and Opus) |
| Streaming | Server-Sent Events (SSE) |
| Containerization | Docker + Docker Compose |

## 📁 Project Structure

```
rag-codebase/
├── app/
│   ├── main.py              # FastAPI - endpoints, sessions, ccc + Anthropic integration
│   ├── config.yml           # repo/branch → local path mapping
│   └── static/
│       └── index.html       # React SPA via CDN (full UI)
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
├── uv.lock
└── .env.example             # required environment variables
```

## 📦 Dependencies

External dependencies required before executing the project.

### Python
```bash
sudo apt update
sudo apt install python3 python3-pip pipx
```

### cocoindex-code
```bash
# Install cocoindex-code
pipx install 'cocoindex-code[full]==0.2.34'

# Create the '~/.cocoindex_code/global_settings.yml' configuration file
ccc init

# Reset local settings
ccc reset --all

# Run diagnostics
ccc doctor
```

### Embeddings Model

File `~/.cocoindex_code/global_settings.yml`:
```yaml
model: nomic-ai/CodeRankEmbed  # trained specifically for source code
device: cuda                   # if omitted, auto-detected (CPU as fallback)
query_params:
    prompt_name: query
indexing_params:
    prompt_name: document
```

> If indexing fails with `MDB_MAP_FULL: Environment mapsize limit reached`, increase the LMDB map size via the `envs: COCOINDEX_LMDB_MAP_SIZE: "size_in_bytes"` environment variable. The value must be a multiple of the system page size (using multiples of 1 MiB is always safe). Examples: 10 GiB (`10737418240`), 20 GiB (`21474836480`), 30 GiB (`32212254720`).

> The similarity threshold `MIN_SCORE_THRESHOLD` and the proportional chunk selection percentages (`CHUNK_SELECTION_TIERS`) defined in `main.py` were tuned based on the scores produced by this model. Changing the embeddings model requires recalibrating these parameters.

## 🔧 Configuration

### Clone

```bash
# Clone the repository and set up the [git-hooks] submodule
git clone https://dev.azure.com/grupo-siagri/ia/_git/rag-codebase
cd rag-codebase && chmod +x getting-started.sh && ./getting-started.sh
```

### Environment Variables

Copy `.env.example` to `.env` and fill in:

- Without Docker (development):
```env
ANTHROPIC_API_KEY=sk-ant-...
CODEBASE_ROOT=/opt/rag/codebase
DAEMON_STOP_ON_CONTEXT_SWITCH=true
```

- With Docker (production):
```env
ANTHROPIC_API_KEY=sk-ant-...
CODEBASE_ROOT=/codebase
CODEBASE_HOST_PATH=/opt/rag/codebase
```

> `docker-compose.yml` bind-mounts `CODEBASE_HOST_PATH` directory into `CODEBASE_ROOT`.

> `DAEMON_STOP_ON_CONTEXT_SWITCH`: When set to `true`, stops the `cocoindex-code` daemon whenever the repository or branch changes, immediately releasing the allocated VRAM. The daemon is automatically restarted during the next warmup. This setting is recommended for development environments with limited GPU resources, whether using `uv` or Docker (the stop and warmup operations affect only the container daemon, without impacting the host daemon responsible for indexing). In production environments with multiple users, leave this setting undefined or set it to `false`, as a context switch by one user would interrupt searches for all other users.

### Repository Mapping

Available repositories and branches are configured in [config.yml](/app/config.yml). Each repository/branch combination corresponds to an independent Git clone with its own `cocoindex-code` index.

> The update policy (`git pull`) and reindexing (`ccc index`) of repositories is managed by external automated schedules, outside this project.

## 💻 Running Locally (WSL/Ubuntu)

### Prerequisites

- Python 3.11+ (`python3 --version`)
  - uv (`uv --version`)
  > To install `uv`, you must run the command: `curl -LsSf https://astral.sh/uv/install.sh | sh`.

### Commands

```bash
# Start the server
uv run uvicorn app.main:app --reload --port 8000
```

Access at `http://localhost:8000`.

## 🚀 Production Server Deployment (Docker)

### Prerequisites

- Docker (`docker --version`) and Docker Compose (`docker compose version`) installed
- Port 8000 open (or configure a reverse proxy via Nginx/Caddy)

### Commands

```bash
# Build and run
docker compose up -d

# Logs
docker compose logs -f

# Stop
docker compose down
```

Access at `http://server:8000`.

## 👤 Author

**Alex Ferreira de Almeida**  
Software Engineer  

## 🔒 Disclaimer

This repository contains **public documentation only**.  
The actual source code of the RAG Codebase is private and cannot be published due to internal processes, proprietary integrations, and corporate security policies.

This README exists solely to document the project’s existence, architecture, design principles, and authorship.

## 📝 License

This project is licensed under the [MIT License](LICENSE).

You are free to use, modify, and distribute this project for personal or commercial purposes, provided that the original authorship and license notice are preserved.

## 📅 Project Status

**Active - 2026 to Present**  
