# Ramanujan

### A production-grade, verifiable AI research engine

### Built to survive the IIITB B.Tech CS workload using LangGraph · Next.js · FastAPI

> **I got tired of ChatGPT hallucinating fake citations and breaking my C code, so I built a multi-agent system that actually searches the web, reads the docs, and cites its sources.**

[Quick Start](#-quick-start) •
[Architecture](#-architecture) •
[Agents](#-agent-types) •
[API Reference](#-api-reference) •
[Configuration](#-configuration) •
[Evaluation](#-evaluation) •
[Contributing](#-contributing)

---

## Table of Contents

* [What Is This?](#what-is-this)
* [Key Features](#key-features)
* [Architecture](#architecture)
* [Project Structure](#project-structure)
* [Quick Start](#quick-start)
  * [Prerequisites](#prerequisites)
  * [Installation](#installation)
  * [Environment Setup](#environment-setup)
  * [Running Locally](#running-locally)
  * [Running with Docker](#running-with-docker)
* [Agent Types](#agent-types)
  * [ReACT Agent](#1-react-agent)
  * [Reflexion Agent](#2-reflexion-agent)
  * [ReWOO Agent](#3-rewoo-agent)
  * [Orchestrator Agent](#4-orchestrator-agent)
  * [Tree Search Agent](#5-tree-search-agent)
* [Workflows](#workflows)
* [Tools](#tools)
* [Multi-Agent Systems](#multi-agent-systems)
* [API Reference](#api-reference)
* [Configuration](#configuration)
* [Evaluation](#evaluation)
* [Observability](#observability)
* [Testing](#testing)
* [Deployment](#deployment)
* [Roadmap](#roadmap)
* [Contributing](#contributing)
* [License](#license)

---

## What Is This?

**Ramanujan** is a production-ready AI research assistant engineered for rigorous academic environments.

When you are managing a heavy Computer Science workload, raw AI models are confident liars. They fail in three specific, highly frustrating ways:

1. **The Hallucinated Citation:** When writing an opinion piece for Technical Communication (TC), standard LLMs confidently generate perfectly formatted APA 7th edition citations for papers and DOIs that literally do not exist.
2. **Surface-Level Fluff:** If you are researching a nuanced sociological topic—like plurilingualism in digital governance for a Social Pathways to Information Technology (SPIT) debate—a standard prompt returns generic, shallow talking points.
3. **Fake Documentation:** When building a socket-based client-server architecture in C for OS Lab (EGC 301P), ChatGPT frequently hallucinates function parameters for concurrency control or Write-Ahead Logging that break the compiler.

### The Solution

Unlike a raw LLM, this agent never makes up facts. You ask a question in natural language. The agent:

1. **Plans** how to answer it (which strategy, how many steps)
2. **Searches** the live web using Tavily or SerpAPI
3. **Scrapes** relevant documentation and papers for detailed content
4. **Reasons** step-by-step using one of five agent strategies
5. **Verifies** its own answer through self-critique (Reflexion)
6. **Synthesizes** a final, cited, markdown-formatted answer
7. **Streams** the result token-by-token to the Next.js client

---

## Key Features

### Five Agent Strategies

Choose automatically via smart routing or manually per request:

* **ReACT** — Fast, iterative reason-and-act loops for quick definitions.
* **Reflexion** — ReACT + self-critique and automatic revision for essay drafting.
* **ReWOO** — Full plan upfront, parallel execution, single synthesis.
* **Orchestrator** — Decomposes complex systems architecture queries into parallel sub-agents.
* **Tree Search** — Explores multiple reasoning paths for ambiguous problems.

### Production Tool Stack

* **Web Search** — Tavily (primary) or SerpAPI (fallback)
* **Web Scraper** — Playwright + BeautifulSoup, cleans boilerplate
* **MCP Support** — Connect any Model Context Protocol server (e.g., local codebase indexing)

### The Study Dashboard

* **Next.js 14 UI** — A dedicated web interface with Tailwind CSS.
* **Live Agent Terminal** — Watch the worker agents in real-time as they spawn, search, scrape, and critique.
* **Receipts Sidebar** — A persistent sidebar aggregating every URL the agents scraped.

### Production Infrastructure

* **REST API** — FastAPI with full OpenAPI docs
* **Redis Cache** — SHA256-keyed response caching (prevents redundant scraping for overlapping syllabus topics)
* **Rate Limiting** — Per-IP sliding window
* **Structured logging** — structlog + JSON in production

---

## Architecture

### System Overview

```text
                        ┌─────────────────────────────────┐
                        │      Next.js Dashboard UI       │
                        │    (Live Terminal + Sidebar)    │
                        └──────────────┬──────────────────┘
                                       │
                        ┌──────────────▼──────────────────┐
                        │         FastAPI (REST)          │
                        │  middleware: rate limit, errors │
                        └──────────────┬──────────────────┘
                                       │
                        ┌──────────────▼──────────────────┐
                        │          Redis Cache            │
                        │   (SHA256 keyed, 1hr TTL)       │
                        └──────────────┬──────────────────┘
                                 miss  │
                        ┌─────────────▼───────────────────┐
                        │         Query Router            │
                        │  rule-based pre-filter +        │
                        │  LLM-based classification       │
                        └──┬───────┬──────┬──────┬────────┘
                           │       │      │      │
              ┌────────────▼─┐ ┌───▼──┐ ┌▼────┐ ┌▼──────────────┐
              │  ReACT Agent │ │ReWOO │ │Refl.│ │  Orchestrator │
              │  (fast Q&A)  │ │Agent │ │Agent│ │  (multi-part) │
              └──────┬───────┘ └──┬───┘ └──┬──┘ └──────┬────────┘
                     │            │        │           │
              ┌──────▼────────────▼─────────▼────────────▼───────┐
              │                 Tool Executor                    │
              │           (parallel or sequential)               │
              └───┬──────────┬──────────┬──────────┬─────────────┘
                  │          │          │          │
           ┌──────▼──┐ ┌─────▼───┐ ┌───▼────┐ ┌──▼──────────┐
           │   Web   │ │  Web    │ │ Calc-  │ │     MCP     │
           │ Search  │ │ Scraper │ │ ulator │ │   Servers   │
           └─────────┘ └─────────┘ └────────┘ └─────────────┘
```

### Agent Decision Flow

```text
User Query
    │
    ▼
┌───────────────────────────────────────────────┐
│              TaskPlanner                      │
│   Analyzes complexity → PlanningLevel (1-5)   │
└───────────────────────┬───────────────────────┘
                        │
          ┌─────────────▼──────────────┐
          │        QueryRouter         │
          │  Rule-based quick classify │
          │  ────────────────────────  │
          │  LLM-based deep classify   │
          └─────┬──────┬──────┬───────┘
                │      │      │
     ┌──────────▼─┐  ┌─▼────┐ ┌▼───────────────────┐
     │  simple_qa │  │ calc │ │  research /        │
     │  → ReACT   │  │→ReACT│ │  multi_faceted /   │
     └────────────┘  └──────┘ │  → Reflexion /     │
                              │  → Orchestrator    │
                              └─────────────────────┘
```

---

## Project Structure

```text
project_ramanujan/
│
├── 📄 docker-compose.yml          # Agent + Redis + Prometheus
├── 📄 README.md                   # This file
│
├── 📁 frontend/                   # Next.js Study Dashboard
│   ├── app/                       # App Router pages
│   ├── components/                # Terminal, Sidebar, UI components
│   └── package.json               # Frontend dependencies
│
└── 📁 backend/                    # Core AI Engine
    ├── 📄 pyproject.toml          # Dependencies, tool settings
    ├── 📄 .env.example            # Environment variables
    ├── 📁 api/                    # FastAPI application & routes
    ├── 📁 agents/                 # ReACT, Reflexion, Orchestrator implementations
    ├── 📁 core/                   # LLM clients, token management
    ├── 📁 tools/                  # Web search, Playwright scraper, MCP
    ├── 📁 workflows/              # Routing and parallelization patterns
    └── 📁 tests/                  # Pytest test suite
```

---

## Quick Start

### Prerequisites

| Requirement | Version | Notes |
| --- | --- | --- |
| Python | 3.11+ | Uses `match` statements |
| Node.js | 18+ | For Next.js frontend |
| Redis | 7+ | For response caching |
| Docker | 24+ | Optional, for containerized run |
| OpenAI API Key | — | Primary LLM provider |
| Tavily API Key | — | Primary search provider |

---

### Installation

#### 1. Start the Reasoning Engine (Backend)

```bash
git clone https://github.com/TanmayDixit/project-ramanujan.git
cd project-ramanujan/backend

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Install Playwright browser (for web scraping)
playwright install chromium
```

#### 2. Environment Setup

```bash
cp .env.example .env
```

Open `.env` and set:

```env
OPENAI_API_KEY=sk-proj-...
TAVILY_API_KEY=tvly-...
REDIS_URL=redis://localhost:6379/0
```

#### 3. Start the Next.js Dashboard (Frontend)

Open a new terminal window:

```bash
cd project-ramanujan/frontend
npm install
```

---

### Running Locally

```bash
# Terminal 1: Start Redis
docker run -d -p 6379:6379 redis:7-alpine

# Terminal 2: Start the FastAPI server
cd backend
uvicorn api.main:app --host 0.0.0.0 --port 8000 --reload

# Terminal 3: Start the UI
cd frontend
npm run dev
```

Navigate to `http://localhost:3000` to access the dashboard.
API docs are at `http://localhost:8000/docs`.

---

### Running with Docker

```bash
# Start everything: frontend, backend, Redis, Prometheus
docker-compose up -d

# View logs
docker-compose logs -f backend
```

---

## Agent Types

### 1. ReACT Agent

**Best for:** Quick lookups, algorithm documentation, current events.
**The Pitch:** Rapidly searches the web, reads the top results, and gives you the exact, grounded summary without the fluff. Perfect for finding Kahn's Algorithm edge cases.

**Configuration:**

```json
{
  "query": "What is the optimal data structure for Dijkstra's algorithm?",
  "agent_type": "react"
}
```

### 2. Reflexion Agent

**Best for:** Academic essays, deep-dive research, when accuracy is non-negotiable.
**The Pitch:** Acts as a harsh grader. It writes a draft, critiques its own claims against the scraped evidence, and re-searches the web if it detects weak evidence or missing citations.

**Configuration:**

```json
{
  "query": "Analyze plurilingualism in digital governance.",
  "agent_type": "reflexion"
}
```

### 3. ReWOO Agent

**Best for:** Queries with a clear, known sequence of research steps. Most token-efficient.
**The Pitch:** Plans the entire execution upfront, runs non-dependent tool calls in parallel, and synthesizes once.

### 4. Orchestrator Agent

**Best for:** Massive assignments, complex system comparisons.
**The Pitch:** Breaks a massive prompt down into sub-tasks, deploys parallel worker agents to research each part simultaneously, and stitches together a comprehensive, cited report.

**Configuration:**

```json
{
  "query": "Compare physical storage query optimization between B-Trees and B+ Trees.",
  "agent_type": "orchestrator"
}
```

### 5. Tree Search Agent

**Best for:** Ambiguous debugging, hypothesis generation.
**The Pitch:** Uses Beam Search over reasoning paths to explore multiple potential solutions before committing to an answer.

---

## Workflows

Workflows are reusable reasoning patterns.

### Prompt Chaining

Execute a sequence of LLM calls where each step's output feeds the next.

```python
from workflows.prompt_chaining import PromptChain, ChainStep
from core.llm_client import LLMClient

chain = PromptChain(llm_client=LLMClient())
chain.add_step(ChainStep(name="identify_intent", ...))
chain.add_step(ChainStep(name="generate_queries", ...))
```

### Routing

Route queries to different handlers based on LLM classification.

```python
from workflows.routing import QueryRouter, QueryClassifier, Route

router = QueryRouter(llm_client=llm)
router.add_route(Route("simple_qa", "Short factual Q&A", react_handler))
router.add_route(Route("research", "Deep analysis needed", reflexion_handler))
```

---

## Tools

### Built-in Tools

| Tool | Name | Description |
| --- | --- | --- |
| Web Search | `web_search` | Search via Tavily or SerpAPI |
| Web Scraper | `scrape_webpage` | Playwright + BeautifulSoup scraper |
| Calculator | `calculator` | Safe sandboxed math eval |

### MCP Integration

Connect any Model Context Protocol server:

```python
from tools.mcp_client import MCPRegistry, MCPServerConfig
from tools import build_default_registry

mcp = MCPRegistry()
mcp.add_server(MCPServerConfig(name="local_codebase", base_url="http://localhost:3001"))
```

---

## Multi-Agent Systems

### Orchestrator-Worker Pattern

One orchestrator LLM decomposes the query; N worker ReACT agents run in parallel.

```python
agent = build_agent(
    agent_type=AgentType.ORCHESTRATOR,
    max_workers=4,
)
state = await agent.run("Compare memory management in Linux, Windows, and macOS")
```

### A2A Protocol

Expose any agent as an A2A-compliant HTTP service to call remote agents.

---

## API Reference

### Endpoints

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/v1/health` | Health check + provider status |
| `POST` | `/v1/ask` | Submit query (batch response) |
| `POST` | `/v1/ask/stream` | Submit query (SSE streaming) |
| `POST` | `/v1/evaluate` | Evaluate answer quality |

### Streaming (SSE)

The `/v1/ask/stream` endpoint uses Server-Sent Events to power the Next.js live terminal.

```python
import httpx

async with httpx.AsyncClient() as client:
    async with client.stream("POST", "http://localhost:8000/v1/ask/stream", json={"query": "WAL recovery sequence"}) as resp:
        async for line in resp.aiter_lines():
            # Process token stream
```

---

## Configuration

Settings are loaded via environment variables:

| Variable | Description | Default |
| --- | --- | --- |
| `OPENAI_API_KEY` | Primary LLM provider | — |
| `TAVILY_API_KEY` | Primary search provider | — |
| `REDIS_URL` | Redis connection | `redis://localhost:6379/0` |
| `MAX_AGENT_ITERATIONS` | Max reasoning loops | `10` |
| `RATE_LIMIT_REQUESTS` | Requests per window | `100` |

---

## Evaluation

### Answer Quality Metrics

Answers are evaluated using LLM-as-Judge across multiple dimensions:

* `factual_accuracy` (30%)
* `completeness` (20%)
* `hallucination_risk` (20%)
* `source_usage` (15%)

Run built-in benchmarks:

```bash
python -m evaluation.benchmarks
```

---

## Observability

### Prometheus Metrics

Exposed at `/metrics`. Key metrics include:

* `http_request_duration_seconds`
* `agent_iterations_total`
* `cache_hits_total`

### Structured Logging

In `production`, logs are formatted as JSON for ELK/Datadog ingestion.

---

## Testing

Mocked LLM client and HTTP calls ensure fast, robust testing without API costs.

```bash
# All tests
pytest tests/ -v

# Enforce coverage
pytest --cov=. --cov-fail-under=80
```

---

## Deployment

### Docker Production Deploy

```bash
docker build --target runtime -t project-ramanujan-api:v1.0.0 .
docker run -d --env-file .env.production -p 8000:8000 project-ramanujan-api:v1.0.0
```

### Gunicorn

```bash
gunicorn api.main:app --worker-class uvicorn.workers.UvicornWorker --workers 4 --bind 0.0.0.0:8000
```

---

## Roadmap

- [ ] **Persistent memory** — Cross-session conversation history via Redis.
- [ ] **PDF ingestion** — Upload and query assignment documents directly.
- [ ] **Vector store integration** — RAG over textbook knowledge bases.
- [ ] **Self-improving agents** — Agents that update their own system prompts based on evaluation metrics.

---

## Contributing

1. Fork the repository
2. Write tests first (coverage ≥ 80%)
3. Run pre-commit hooks (`ruff`, `mypy`)
4. Open a Pull Request

---

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## Contact

* **Author:** Tanmay Dixit
* **Education:** B.Tech Computer Science, IIIT Bangalore (Class of 2028)
* **GitHub:** [TanmayDixit](https://github.com/TanmayDixit)
