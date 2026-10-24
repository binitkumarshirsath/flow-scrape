# Build Log

Append one entry per session. Newest at the bottom.

## Template
### YYYY-MM-DD — Milestone / Week / Day
- **Did:**
- **Learned:**
- **Stuck on / questions:**
- **Next:**
- **Interview Q answered:**

---

### 2026-10-24 — Planning
- **Did:** Chose Flow Scrape (visual web-scraper + AI) with Next.js + FastAPI. Wrote CLAUDE.md and PLAN.md.
- **Next:** M0 Day 1 — install tools, create repo and folder layout.

### 2026-10-24 — M0 / Week 1 / Day 1
- **Did:** Installed/verified tools: Node 24 (upgraded from EOL Node 20), npm 11, uv 0.12, Docker 28. Created `api/` folder. Updated PLAN: layered backend (router → service → repository), skills checklist, fixed `api/app/` tree, added tooling to Day 3. Added CLAUDE.md rule #8 (predict before building, quiz after).
- **Learned:** 
  - pip installs globally by default, so we need a venv (.venv/), a per-project package folder like node_modules/, so projects needing different versions don't clash. The lockfile comes from uv (uv.lock), not from the venv.
  - Python installs packages globally by default; a virtual environment (`.venv/`) is a per-project package folder ≈ `node_modules/` + which Python to use. Never commit it.
  - pip = basic installer, no real lockfile (`requirements.txt` is hand-maintained). uv = pnpm + nvm in one: manages Python versions, creates `.venv`, `pyproject.toml` ≈ `package.json`, `uv.lock` ≈ lockfile.
  - Many Pythons can coexist on one machine (system, Homebrew, python.org, Anaconda); uv pins one per project like `.nvmrc`. Chose 3.13 (stable, full library support).
  - npm vs pnpm: pnpm = shared store (fast, less disk) + strict `node_modules` (no phantom dependencies). Sticking with npm.
- **Stuck on / questions:** none
- **Next:** M0 Day 2 — scaffold `web/` (Next.js + TS + Tailwind + shadcn) and run it.
- **Interview Q answered:** none yet

### 2026-10-24 — M0 / Week 1 / Day 2
- **Did:** Scaffolded `web/` with create-next-app: Next.js 16.4, React 19.3, App Router, TypeScript (strict), Tailwind v4, ESLint, npm. Initialized shadcn/ui (`components.json`, `Button`, `cn()` in `lib/utils.ts`). Build + lint pass.
- **Learned:** Next 16 generates `AGENTS.md` (notes for AI tools about API changes); commit it or `next dev` keeps re-creating it.
- **Stuck on / questions:** none
- **Next:** M0 Day 3 — `api/`: `uv init`, pin Python 3.13, FastAPI hello world + `/health`, pydantic-settings, ruff + mypy + pre-commit.
- **Interview Q answered:** none yet
