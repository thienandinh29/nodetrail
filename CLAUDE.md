@MAP.md
@SPEC.md
@PLAN.md
@TEAM.md

# NodeTrail

AI tutor for any subject. The user states a goal ("Tôi muốn học Python") or uploads a syllabus;
NodeTrail builds a Knowledge Graph from trusted textbooks and runs the loop
assess → learn → practice → adjust → review. Mascot: **Sena** the monkey.
Full brief: `context/product-brief.vi.md`.

## Owner & working agreement
- Team of 5 (lead: student, aspiring ML engineer), each driving an AI coding agent. Lanes, owned paths,
  contracts and the 24-hour timeline are in `TEAM.md`; stay inside your lane's paths.
- **Current goal: competition demo built in 24 hours** by the 5 lanes in `TEAM.md`: the judge names any topic or
  uploads a file and NodeTrail builds a cited path live. Stay inside your lane and the demo scope in `SPEC.md`;
  if a task grows, propose a cut instead of extending it.
- Even in the sprint, keep the architecture clean (engine pure + tested, LLM behind the gateway) so nothing must be rewritten later.
- **Reply in English.** Detailed answers with the reasoning behind decisions.
- **Ask before** committing, pushing, adding dependencies, or making architectural changes.
- **Run tests/lint before saying anything is done** (see `/verify` skill). Report failures honestly with output.
  The owner re-checks everything afterwards.

## Language rules
- All code, identifiers, comments, commit messages, and dev docs: **English**.
- Product is for **Vietnamese learners**: all user-facing UI text, Sena's dialogue, tutor responses,
  and sample content default to **Vietnamese**. Keep UI strings in i18n files (`vi` default, `en` secondary),
  never hard-coded in components.
- Handle Vietnamese text correctly: UTF-8 everywhere, NFC normalization, diacritic-aware search/tokenization.

## Core architecture principle
LLM is **not** the decision maker. It sits inside a harness:
- **Learning Engine** decides what to learn next: Bayesian Knowledge Tracing, rule engine (graph-walk placement, root-cause remediation), FSRS.
- **Graders** score work objectively: answer keys now (MCQ); test cases and rubrics later.
- **LLM + RAG** only explains, always with citations to source textbooks. No citation → don't present it as fact.
- Socratic tutor never hands over the final answer; it gives step-by-step hints.
- Sena praises only real understanding (mastery), never speed or streaks.

## Stack (decided — details and contracts in `TEAM.md`)
- Web: `apps/web` — Next.js App Router, TypeScript strict, Tailwind + shadcn/ui, React Flow + `dagre` auto-layout; Vercel.
- API: `apps/api` — Python 3.12, FastAPI, Pydantic v2, uv; Docker on a teammate's machine via a named Cloudflare
  Tunnel (`--protocol http2`) on the team's domain; multi-stage images from GHCR, auto-updated by Watchtower.
  Model weights are bind-mounted from the host, never baked into the image.
- Jobs: `topic_jobs` table in Postgres as the queue (`SELECT … FOR UPDATE SKIP LOCKED`); the worker is a separate
  compose service from the same image (Redis after the contest). Never long work inside a request; SSE progress
  with a heartbeat every 15–20 s.
- Data: Supabase — Postgres + pgvector (`halfvec`), Storage; ~30k-chunk budget on the free tier.
  Migrations in `supabase/migrations/`.
- Embeddings: BAAI/bge-m3 (1024 dims) behind `Embedder`. Only the API process loads the model (worker calls
  `/internal/embed`); encode via `asyncio.to_thread`, never blocking the event loop.
- LLM: **OpenRouter only**, through `app/llm/gateway.py`; model per (tier, task) in `app/llm/routing.yaml`.
  Never call an LLM SDK or HTTP API directly from features.
- Observability: structlog JSON logs + Langfuse (OpenTelemetry and Sentry after the contest).

## Code conventions
- Python: full type hints, `ruff` (lint + format) and `pytest` required in CI; `mypy` runs but only warns. Learning-engine math must have unit tests.
- TypeScript: `strict: true`, no `any`, ESLint + Prettier, Vitest for logic.
- Small modules, pure functions for algorithms (BKT/FSRS/Elo) so they are testable without DB or LLM.
- Secrets only in `.env` (never commit, never paste in chat). Document every new variable in `.env.example`.

## Folders
- `context/` — reference material for Claude (brief, research, textbook notes). Read when relevant.
- `outputs/` — generated deliverables (reports, exports, diagrams). Not source code.
