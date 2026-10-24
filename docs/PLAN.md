# Flow Scrape — Project Plan

A visual web-scraper SaaS. Users build workflows by dragging task nodes onto a React Flow canvas (Launch browser → Fill input → Click → Extract text → Webhook), then run them manually or on a cron schedule. Each task costs credits; credits are bought via Stripe. AI nodes extract data and fix selectors.

Based on the "Scrape Flow" YouTube tutorial (Next.js 14 + Prisma + Puppeteer), **re-architected** as Next.js frontend + FastAPI backend, with production fixes and AI enhancements.

---

## 0. What & why (the big picture)

**One-liner:** Flow Scrape turns websites without APIs into reliable, scheduled, structured data feeds that other tools, teams and AI agents can build on.

**Problem:** valuable data lives on websites with no API — behind logins, across pages, changing constantly. Today it's copied by hand (slow, error-prone) or scraped by custom scripts (need devs, break when the site changes).

**Solution:**
- *Accessible* — build automations visually, or generate them from a sentence with AI
- *Reliable* — validated graphs, job-queue execution, AI self-healing selectors
- *Intelligent* — AI extracts structured data; RAG makes scraped history queryable

**User journey:** Build → Validate → Run (live logs) → Publish & schedule → Deliver (webhook) → Ask (chat with data).

**Ecosystem position:**
```
Websites (no API) → Flow Scrape → webhooks (n8n/Zapier) · Slack/email · Sheets ·
                                  data warehouse/BI · AI agents (API / MCP) · RAG knowledge bases
```
- Downstream of websites: the bridge for sites that will never have an API
- Upstream of automation tools: they trigger workflows and consume results
- Upstream of AI: workflows become reliable tools for agents; scraped data becomes a fresh RAG knowledge source (for this app and for external RAG systems)

**Why I'm building it:** to master React Flow, workflow engines/graph algorithms, Python + FastAPI, and AI engineering (structured outputs, RAG, agents, evals) on one real system.

---

## 1. Architecture

```
flow-scrape/
  CLAUDE.md
  docs/PLAN.md
  docker-compose.yml          # postgres, redis
  web/                        # Next.js
  api/                        # FastAPI
```

### web/ (Next.js) — feature-based
```
web/src/
  app/                        # routing ONLY, thin files
    (auth)/sign-in, sign-up
    (dashboard)/
      page.tsx                # home / analytics
      workflows/page.tsx
      credentials/page.tsx
      billing/page.tsx
    workflow/editor/[workflowId]/page.tsx
    workflow/runs/[workflowId]/page.tsx
    workflow/runs/[workflowId]/[executionId]/page.tsx
    setup/page.tsx            # create initial credit balance after sign-up
  features/
    workflows/                # list, card, create/duplicate/delete dialogs
    editor/                   # FlowEditor, NodeCard, handles, TaskMenu, store/, hooks/
    executions/               # RunViewer, PhaseDetails, ExecutionsTable, SSE hook
    credentials/
    billing/
    analytics/
    (each: components/ hooks/ types.ts index.ts)
  components/ui/              # shadcn
  components/layout/          # Sidebar, Breadcrumbs, Logo, ThemeToggle
  lib/                        # cn, dates, api client (generated)
```

### api/ (FastAPI) — layered: router → service → repository
```
api/
  Dockerfile
  pyproject.toml          # uv, ruff, mypy, pytest config
  alembic/                # migrations
  tests/
    unit/                 # engine, services (no DB)
    integration/          # API + real Postgres (testcontainers)
  app/
    main.py                 # app factory, middleware, exception handlers, lifespan
    core/
      config.py             # pydantic-settings (env-based config)
      db.py                 # async engine + session dependency
      security.py           # Clerk JWT verification, encryption
      logging.py            # structured JSON logs + request IDs
      telemetry.py          # OpenTelemetry tracing
      errors.py             # domain exceptions → HTTP responses
      rate_limit.py
    api/v1/                 # routers (HTTP only: parse, call service, return schema)
      workflows.py, executions.py, credentials.py, billing.py, analytics.py, webhooks.py, cron.py
    services/               # business logic (no HTTP, no SQL details)
    repositories/           # DB access (SQLAlchemy queries only)
    models/                 # SQLAlchemy models
    schemas/                # Pydantic request/response models (the API contract)
    engine/
      tasks/       # task definitions: type, label, inputs, outputs, credits
      executors/   # one async function per task type
      registry.py  # TaskType -> (definition, executor)
      plan.py      # flow -> execution plan + validation
      environment.py
      run.py       # execute_workflow
    ai/
      extraction.py         # structured extraction with Pydantic
      html_preprocess.py
      workflow_generator.py # natural language -> workflow
      selector_healer.py
      browser_agent.py
      rag/                  # ingest.py, chunking.py, embeddings.py, retrieve.py, answer.py
      evals/
    workers/     # Arq tasks: run_workflow, cron tick
```

### Key design decisions (be ready to defend these)
1. **Engine is framework-free** — `engine/` has no FastAPI imports, so it's unit-testable and runnable by the worker.
2. **Runs go through a job queue**, not a fire-and-forget request (tutorial's flaw: dies on serverless/restart).
3. **Live updates via SSE**, not 1s polling.
4. **Credits deducted atomically** (`UPDATE ... SET credits = credits - x WHERE credits >= x`) with an idempotency key.
5. **Backend is the source of truth for validation.** Frontend does a light TS check only for UX (red borders); FastAPI re-validates on run/publish.
6. **Typed contract**: OpenAPI → generated TS client.

---

## 2. Data model
- **Workflow**: id, user_id, name, description, definition (JSON: nodes, edges, viewport), status (DRAFT/PUBLISHED), execution_plan (JSON, frozen on publish), credits_cost, cron, next_run_at, last_run_at, last_run_id, last_run_status, created_at, updated_at. Unique (user_id, name).
- **WorkflowExecution**: id, workflow_id, user_id, trigger (MANUAL/CRON/API), status (PENDING/RUNNING/COMPLETED/FAILED), definition (snapshot), credits_consumed, created_at, started_at, completed_at.
- **ExecutionPhase**: id, execution_id, user_id, number, node (JSON), name, status (CREATED/PENDING/RUNNING/COMPLETED/FAILED), inputs, outputs, credits_consumed, started_at, completed_at.
- **ExecutionLog**: id, phase_id, level (INFO/ERROR), message, timestamp.
- **Credential**: id, user_id, name, value (encrypted), created_at. Unique (user_id, name).
- **UserBalance**: user_id (PK), credits.
- **UserPurchase**: id, user_id, stripe_id, description, amount, currency, date.

---

## 3. Phases (tick as you go)

### Phase 0 — Python primer (in parallel with Phase 1–2)
- [ ] Python syntax vs JS: types, dicts/lists, comprehensions, classes, dataclasses
- [ ] async/await in Python, `asyncio.gather` (≈ Promise.all)
- [ ] Pydantic v2 (≈ zod), virtualenv/uv, pytest
- [ ] Build a tiny FastAPI todo API with Postgres

### Phase 1 — Frontend foundation (Claude can help a lot)
- [ ] Next.js 15/16 + TS + Tailwind + shadcn, feature-based folders
- [ ] Clerk auth (v6 middleware), sign-in/up pages, `/setup` redirect after sign-up
- [ ] Dashboard layout: sidebar (desktop + mobile sheet), breadcrumbs, theme toggle, not-found page
- [ ] TanStack Query provider + devtools, top loader
- [ ] Note: in Next 15+ `params`/`searchParams` are **Promises** — `await` them

### Phase 2 — React Flow editor (I write the logic)
- [ ] Editor page + ReactFlowProvider, background, controls, snap grid, fitView
- [ ] Task definitions + registry (typed with generics; key must match task `type`)
- [ ] Custom node: header (icon, label, entry-point badge, credits, drag handle, delete, duplicate), inputs, outputs
- [ ] Param components: string (input/textarea), browser-instance, select, credential
- [ ] Typed, colored handles; hide handle for manual-only inputs
- [ ] Clear input value on connect; disable input when connected; one edge per input
- [ ] `isValidConnection`: no self-connection, type match, **no cycles (DFS)**
- [ ] Deletable custom edge (smooth step path)
- [ ] Drag & drop from TaskMenu (`screenToFlowPosition`), double-click to center node
- [ ] Save/restore definition + viewport
- [ ] **Beyond tutorial**: Zustand flow store, undo/redo, debounced autosave, auto-layout (dagre/elkjs)
- [ ] Interview check: why `nodeTypes` outside component? controlled vs uncontrolled? memo on nodes? 500+ nodes performance?

### Phase 3 — FastAPI backend foundation
- [ ] Project setup (uv/poetry), config, SQLAlchemy + Alembic, docker-compose (postgres, redis)
- [ ] Clerk JWT verification dependency (JWKS)
- [ ] Workflow CRUD + duplicate; generated TS client wired into `web/`
- [ ] Interview check: how does JWKS verification work? why generate the client?

### Phase 4 — Execution engine (core; I write it)
- [ ] `plan.py`: phases algorithm (entry point → repeatedly plan nodes whose incomers are all planned), collect invalid inputs per node, errors `NO_ENTRY_POINT` / `INVALID_INPUTS`
- [ ] Unit tests for the planner (linear, diamond, parallel branches, missing input, unreachable node)
- [ ] Frontend: execute button, highlight invalid nodes/inputs from validation response
- [ ] Environment: per-node inputs/outputs, shared browser + page, resolve inputs from upstream outputs via edges
- [ ] Executors (Playwright): launch browser, page→HTML, extract text (BeautifulSoup), fill input, click, wait for element, navigate URL, scroll to element, read/add JSON property, deliver via webhook
- [ ] Logging collector per phase; persist inputs/outputs/logs/credits/status
- [ ] Atomic credit deduction; stop workflow on insufficient credits
- [ ] Arq job queue: API enqueues, worker runs; cleanup browser in `finally`
- [ ] **Run phases in parallel** inside a phase (`asyncio.gather`) — optional flag
- [ ] SSE endpoint streaming phase status + logs
- [ ] Interview check: complexity of planning? is this topological sort (Kahn's)? how do you handle a crashed worker mid-run?

### Phase 5 — Run viewer & history
- [ ] Run page: status, started, duration, credits (animated), phase list with status icons
- [ ] Phase details: credits, duration, inputs/outputs viewers, logs table
- [ ] Auto-select running phase / last phase
- [ ] Executions history table (trigger, status, credits, started)

### Phase 6 — Publish & schedule
- [ ] Publish: validate, freeze execution plan, compute credits cost; unpublish
- [ ] Workflow card: run button, last run status, next run (local + UTC), schedule dialog (cron validation + human-readable), duplicate, description tooltip
- [ ] Cron: scheduler tick finds due workflows, enqueues runs, updates `next_run_at` (protect endpoint with secret, constant-time compare)
- [ ] Interview check: what if two ticks overlap? (row locking / `SELECT ... FOR UPDATE SKIP LOCKED`)

### Phase 7 — Credentials
- [ ] AES-256-GCM encryption (key from env, random IV, store `iv:ciphertext`) — prefer GCM over CBC and know why
- [ ] Credentials page: create, list, delete
- [ ] Credential param in nodes; decrypt only inside executors

### Phase 8 — Billing & analytics
- [ ] Stripe checkout for 3 credit packs; webhook (verify signature) → upsert balance + purchase record (idempotent on event id)
- [ ] Billing page: balance, purchase packs, monthly usage chart, transaction history, invoice download
- [ ] Home analytics: period selector, stats cards, execution status area chart, credits usage bar chart

---

## 4. AI enhancements (I build these; order = learning ramp)

1. **Structured extraction node** — user defines fields (name/type/description); backend builds a Pydantic model dynamically; LLM returns validated JSON; retry with error feedback on validation failure. *Learn: structured outputs, validation loops.*
2. **HTML preprocessing** — strip scripts/styles/svg, keep visible text + key attributes, chunk large pages, count tokens, log cost per phase. *Learn: token/cost engineering.*
3. **AI selector finder** — given HTML + description ("username field"), return selectors; verify each selector exists on the page before using it. *Replaces the tutorial's "prompt + read JSON property" chain.*
4. **Natural language → workflow** — LLM outputs nodes/edges (structured output against the task registry schema); run `plan.py` validation; feed errors back for repair (max N attempts); show the generated graph in the editor for review. *Learn: deterministic validation of LLM output. Strongest interview story.*
5. **Self-healing selectors** — when click/fill/wait fails, AI inspects the DOM, proposes a new selector, retries once, logs the change and suggests updating the workflow. *Learn: ReAct in practice.*
6. **Browser agent node** — goal + tools (navigate, click, fill, extract, done); step limit; full trace in logs. *Learn: agents, tool calling, guardrails.*
7. **Evals** — dataset of saved HTML pages + expected outputs; score extraction & selector accuracy per prompt/model version; track in a simple table. *Learn: evals — what AI engineering interviews ask about.*
8. **Observability** — log tokens, latency, cost, model per AI call; show in phase details.
9. **RAG: "Chat with your scraped data"** — *Learn: embeddings, chunking, vector search, hybrid search, reranking, grounded answers, RAG evals.*
   - [ ] Enable `pgvector` in Postgres (no new infra)
   - [ ] New tables: `Document` (user_id, workflow_id, execution_id, phase_id, source_url, scraped_at, raw_text) and `Chunk` (document_id, text, embedding vector, metadata JSON)
   - [ ] Ingestion: after a run completes, enqueue an Arq job → chunk outputs (try fixed-size vs structure-aware chunking and compare) → embed → store
   - [ ] Retrieval: vector similarity search filtered by user_id (+ optional workflow/date filters); then add **hybrid search** (Postgres full-text + vector, merged with reciprocal rank fusion)
   - [ ] Optional reranking step (cross-encoder or LLM rerank) on top-k
   - [ ] Answer generation with **citations** to specific runs/URLs; refuse when context doesn't contain the answer
   - [ ] Chat UI page in `web/features/rag-chat/` with streaming answers (SSE) and clickable citations → run viewer
   - [ ] **Knowledge-source mode**: a "Push to knowledge base" task node so any workflow can feed a named RAG collection (internal or external RAG via webhook/API); scheduled re-crawls keep it fresh
   - [ ] Incremental re-indexing: content hash per page/chunk → only re-embed what changed; mark stale/removed docs
   - [ ] Dedup + quality filters (boilerplate removal, near-duplicate detection, min length)
   - [ ] **Prompt-injection defense**: scraped text is untrusted — keep it in clearly delimited context, never as instructions; test with a page containing injected instructions
   - [ ] RAG evals: question/answer/expected-source dataset; measure retrieval hit rate (recall@k) and answer faithfulness
   - [ ] Interview check: why chunk? how pick chunk size? vector vs keyword search — when does each fail? how do you stop hallucinations? how do you evaluate retrieval separately from generation? multi-tenant data isolation in a vector store?

---

## 5. Tutorial deviations (things that will differ from the video)
- Next 14 → 15/16: async `params`/`searchParams`, Clerk v6 middleware, newer shadcn CLI.
- Prisma/SQLite → SQLAlchemy/Postgres in FastAPI.
- Puppeteer → Playwright (Python); Cheerio → BeautifulSoup.
- Server actions for data → FastAPI endpoints via generated client (server actions only as thin BFF where useful).
- Polling → SSE. Fire-and-forget execution → job queue.
- `getIncomers` from @xyflow doesn't exist server-side → implement incomers yourself in Python.

---

## 6. Schedule — 1 hour/day

**Strategy: thin vertical slice first.** Get a tiny end-to-end version working (editor → FastAPI → planner → Playwright run → one AI node) by ~week 8, then deepen each layer. Early demo = motivation; every later phase upgrades something that already works.

**Daily session (60 min):**
- 5 min — read `docs/LOG.md` last entry + current milestone below
- 45 min — build (one small task)
- 10 min — commit, append to `docs/LOG.md`: what I did, what I learned, what's next, one interview question answered in my own words

**Weekly rhythm:** Mon–Fri build. Sat: Python learning during weeks 1–6, afterwards review/refactor/interview Q&A. Sun: rest or catch-up.

**Rough total: ~7–8 months.** Billing & analytics are optional and last.

### M0 — Setup (week 1)
- Day 1: install tools, create repo + folder layout, add CLAUDE.md + docs/PLAN.md + docs/LOG.md, first commit
- Day 2: `web/` — Next.js + TS + Tailwind + shadcn init, run it
- Day 3: `api/` — uv project (pin Python 3.13), FastAPI hello world, `/health` endpoint, pydantic-settings config, ruff + mypy + pre-commit
- Day 4: docker-compose with Postgres + Redis (local dev), connect FastAPI to Postgres
- Day 5: Clerk in `web/`, sign-in/up pages, protected dashboard route
- Sat: Python basics (types, lists/dicts, functions, classes)

### M1 — Thin vertical slice (weeks 2–8)
- W2: dashboard layout (sidebar, breadcrumbs, theme); Workflow model + Alembic migration; Sat: Python async + Pydantic
- W3: Clerk JWT verification in FastAPI; workflow CRUD endpoints; generate TS client (openapi-typescript); workflows list page; Sat: FastAPI deps + SQLAlchemy basics
- W4: editor page with React Flow; task registry (TS) with 3 tasks: Launch browser, Get HTML, Extract text; custom node card with inputs/outputs
- W5: string param input, typed handles, connections, save/load definition via API
- W6: Python: task registry mirror + `plan.py` (phases + validation) + pytest tests for planner
- W7: executors with Playwright (3 tasks), environment object, run endpoint (synchronous for now), store execution + phases; Sat: Playwright basics
- W8: minimal run page (phase list + outputs); **AI item 1: structured extraction node** → 🎉 first demo

### M2 — Editor depth (weeks 9–12)
- W9: `isValidConnection` (self, type match), clear/disable connected inputs, one edge per input
- W10: cycle detection (DFS), deletable custom edge, drag & drop from TaskMenu
- W11: move flow state to Zustand; delete/duplicate node; double-click center; frontend validation highlighting
- W12: undo/redo, debounced autosave, auto-layout (dagre/elkjs)

### M3 — Engine to production grade (weeks 13–16)
- W13: Arq job queue + worker; run endpoint enqueues; browser cleanup in `finally`
- W14: log collector per phase; persist inputs/outputs/logs; SSE stream of status + logs
- W15: credits: UserBalance, atomic deduction, insufficient-credit handling; parallel execution within a phase
- W16: full run viewer (phase details, logs table, auto-select phase) + executions history table; remaining executors (fill, click, wait, navigate, scroll, JSON read/add, webhook)

### M4 — AI core (weeks 17–21)
- W17: HTML preprocessing + token/cost logging (AI items 2, 8)
- W18: AI selector finder with on-page verification (item 3)
- W19–20: natural language → workflow with validation + repair loop (item 4)
- W21: self-healing selectors (item 5)

### M5 — Platform (weeks 22–24)
- W22: publish/unpublish, frozen plan, credits cost; workflow card upgrades
- W23: cron scheduling (dialog, scheduler tick, `SKIP LOCKED`), duplicate workflow
- W24: credentials (AES-GCM), credentials page, credential param

### M6 — Agents + RAG (weeks 25–30)
- W25–26: browser agent node (item 6)
- W27: pgvector, Document/Chunk tables, ingestion job, chunking
- W28: vector + hybrid search, reranking
- W29: chat UI with streaming + citations; knowledge-source node; incremental re-index
- W30: prompt-injection defense + dedup/quality filters

### M7 — Evals, deploy, showcase (weeks 31–33)
- W31: eval datasets + scoring (extraction, selectors, RAG) (item 7)
- W32: deploy (web on Vercel, api + worker + Postgres + Redis on Railway/Fly/Render)
- W33: README with architecture diagram, 2-min demo video, write-up of metrics

### M8 — Optional
- Stripe billing, analytics dashboard, setup page, mobile sidebar polish

---

## 7. Skills checklist — what I must be able to build AND explain by the end

Don't treat this as extra work: each item is attached to a week that already exists. Tick only when I can explain it in an interview without notes.

### Production FastAPI backend
- [ ] Layered architecture: router → service → repository; why each layer exists (W3)
- [ ] Dependency injection with `Depends` (db session, current user, settings) (W3)
- [ ] Config with pydantic-settings, 12-factor env config (M0)
- [ ] Async SQLAlchemy 2, Alembic migrations, connection pooling, N+1 queries (W2–W3)
- [ ] API versioning (`/api/v1`), pagination, consistent error format, domain exceptions (W3)
- [ ] Auth: JWT/JWKS verification, authorization (ownership checks), multi-tenancy by user_id (W3)
- [ ] Testing pyramid: pytest, fixtures, unit vs integration, testcontainers, mocking LLM calls (W6 onward)
- [ ] Code quality: ruff, mypy, pre-commit hooks (M0)
- [ ] CI: GitHub Actions running lint + types + tests on every push (W3)
- [ ] Docker: multi-stage builds, docker-compose for local dev (M0, W32)
- [ ] Background jobs: Arq/Redis, retries, idempotency, dead jobs (W13)
- [ ] Real-time: SSE vs WebSockets vs polling — when to use which (W14)
- [ ] Concurrency: async/await, `asyncio.gather`, semaphores to limit browsers (W15)
- [ ] Transactions, row locking, `SELECT … FOR UPDATE SKIP LOCKED`, race conditions (W15, W23)
- [ ] Caching with Redis (cache-aside, TTL, invalidation) (W17 — cache LLM/HTML results)
- [ ] Rate limiting (token bucket) per user/API key (W22)
- [ ] Observability: structured logs, request IDs, OpenTelemetry traces, metrics, health/readiness checks (W14, W32)
- [ ] Security: encryption at rest (AES-GCM), secrets management, webhook signature verification, CORS, input validation, SSRF protection (users submit URLs!) (W24, M8)
- [ ] Deployment: separate API + worker services, migrations on deploy, zero-downtime basics (W32)
- [ ] **Microservices — honest note:** this is a *modular monolith + worker*, which is what most real teams should start with. Be able to explain: when you'd split into services (e.g. an AI service, a scraper-runner service), how they'd communicate (HTTP vs queue/events), and the costs (network failures, distributed tracing, data ownership). Optional stretch (after M7): extract the AI layer into its own FastAPI service talking over the queue.

### AI engineering
- [ ] **LLM basics:** tokens, context window, temperature/top-p, system/user/assistant roles, streaming, cost & latency trade-offs (W8)
- [ ] **Structured outputs:** JSON mode vs schema-constrained outputs, Pydantic validation, retry-with-feedback (W8)
- [ ] **Tool / function calling** (W18)
- [ ] **Prompt engineering & management:** few-shot, chain-of-thought, prompt templates, prompt versioning (W18–W20)
- [ ] **Context engineering:** what to put in context, HTML cleaning, chunking, summarisation, token budgets (W17)
- [ ] **Prompt chaining & validation loops** — LLM output checked by deterministic code (W19–W20)
- [ ] **ReAct, agents:** agent loop, tools, memory, step limits, guardrails, human-in-the-loop (W21, W25–26)
- [ ] **Embeddings & vector search:** cosine similarity, ANN indexes (HNSW/IVFFlat in pgvector), embedding model choice (W27)
- [ ] **RAG:** chunking strategies, hybrid search (BM25 + vector), RRF, reranking, metadata filtering, citations, query rewriting (W27–W29)
- [ ] **Guardrails & safety:** prompt injection, output validation, PII handling, refusals (W30)
- [ ] **Evals:** golden datasets, offline vs online evals, LLM-as-judge (and its biases), retrieval metrics (recall@k, MRR), faithfulness, regression tracking (W31)
- [ ] **LLMOps / observability:** tracing LLM calls (e.g. Langfuse), token/cost tracking, latency, model versioning (W17, W31)
- [ ] **Reliability:** retries with backoff, timeouts, model fallbacks/routing (cheap model first), semantic caching (W17–W20)
- [ ] **MCP:** expose workflows as tools for AI agents (stretch, after M7)
- [ ] **Know the vocabulary (theory, no build needed):** fine-tuning vs RAG vs prompting, LoRA, quantization, distillation, hallucination, transformers/attention at a high level (Karpathy side track)

### Exit test
At the end I should be able to, without notes:
1. Draw the full architecture on a whiteboard and walk through one run end to end.
2. Explain every folder in `api/` and why the layers exist.
3. Explain the RAG pipeline and how I evaluated it, with real numbers.
4. Explain one production incident/bug I hit and how I debugged it (keep notes in LOG.md!).

---

## 8. Interview question bank (answer these in your own words when done)
- Walk me through what happens when a user clicks Execute, end to end.
- How do you detect cycles? Complexity?
- Why phases instead of a flat topological order?
- How are outputs of one node passed to the next?
- What can't be serialized in the environment, and how do you handle it?
- How do you prevent double-charging credits? Two concurrent runs?
- How would you scale to 10k concurrent runs?
- Why is the published workflow's plan frozen?
- How do you securely store and use user API keys?
- How do you make LLM output reliable? How do you know your prompt change made things better?
- Explain your RAG pipeline end to end. Where can it fail, and how did you measure each stage?
- React Flow: how does it render, what triggers re-renders, how do you optimize large graphs?