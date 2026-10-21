# MAP — where things are

Keep this file current when adding top-level folders or important modules.
Planned entries are marked _(planned)_ until they exist.

| Path | Purpose |
|---|---|
| `CLAUDE.md` | Rules and working agreement for Claude (auto-loaded) |
| `SPEC.md` | What NodeTrail must do: features, behaviors, non-goals |
| `PLAN.md` | Phased build plan and current status |
| `context/product-brief.vi.md` | Original product brief (Vietnamese) |
| `context/` | Reference material: research, textbook notes, algorithm papers |
| `outputs/` | Generated deliverables, not source code |
| `.claude/settings.json` | Pre-approved commands for Claude Code |
| `.claude/skills/verify/` | `/verify` — run all lint, type checks, tests |
| `.env.example` | Every env var the project uses (copy to `.env`) |
| `apps/api/` _(planned)_ | FastAPI backend: learning engine, graders, RAG, LLM gateway |
| `apps/api/app/engine/` _(planned)_ | BKT, FSRS, Elo, rule engine — pure, tested |
| `apps/api/app/graders/` _(planned)_ | Per-subject objective graders |
| `apps/api/app/rag/` _(planned)_ | Ingestion, chunking, embeddings, retrieval with citations |
| `apps/api/app/llm/` _(planned)_ | Provider-agnostic LLM interface |
| `apps/web/` _(planned)_ | Next.js frontend, Sena UI, Knowledge Graph view |
| `apps/web/messages/` _(planned)_ | i18n strings (`vi` default, `en`) |
| `supabase/migrations/` _(planned)_ | Database schema (Postgres + pgvector) |
