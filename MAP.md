# MAP — where things are

Keep this file current when adding top-level folders or important modules.
Planned entries are marked _(planned)_ until they exist.

| Path | Purpose |
|---|---|
| `CLAUDE.md` | Rules and working agreement for Claude (auto-loaded) |
| `SPEC.md` | What NodeTrail must do: features, behaviors, non-goals |
| `PLAN.md` | Phased build plan and current status |
| `TEAM.md` | 24-hour team plan: lanes, owned paths, contracts, timeline, git workflow |
| `context/product-brief.vi.md` | Original product brief (Vietnamese) |
| `context/` | Reference material: research, textbook notes, algorithm papers |
| `outputs/` | Generated deliverables, not source code |
| `.claude/settings.json` | Pre-approved commands for Claude Code |
| `.claude/skills/verify/` | `/verify` — run all lint, type checks, tests |
| `.env.example` | Every env var the project uses (copy to `.env`) |
| `docker-compose.yml` | Local Postgres 16 + pgvector on port 54322 (`docker compose up -d`) |
| `.github/workflows/ci.yml` | CI: ruff, mypy, pytest against a pgvector service |
| `apps/api/` | FastAPI backend (uv, Python 3.12): `app/main.py` app factory + `/health` |
| `apps/api/app/config.py` | Settings from repo-root `.env` (pydantic-settings) |
| `apps/api/app/migrate.py` | Applies `supabase/migrations/*.sql` once each (`uv run python -m app.migrate`) |
| `apps/api/tests/` | pytest; `conftest.py` makes a throwaway migrated DB per run |
| `apps/api/app/engine/` | Pure, tested decision logic: `bkt.py` (mastery), `srs.py` (FSRS review scheduling); rule engine planned |
| `apps/api/app/graders/` _(planned)_ | Per-subject objective graders |
| `apps/api/app/rag/` | `notebook.py` (ipynb → chunks), `embed.py` (bge-m3, optional `embed` extra), `store.py`, `ingest.py` CLI; retrieval planned (Day 4) |
| `data/raw/thinkpython/` | Downloaded Think Python 3e notebooks + `SOURCE_SHA` (git-ignored; re-download to rebuild) |
| `apps/api/app/llm/` _(planned)_ | Provider-agnostic LLM interface |
| `apps/demo/` _(planned, sprint)_ | Streamlit demo UI; talks to the API over HTTP only |
| `data/kg/` _(planned)_ | Reviewed Knowledge Graph YAML (concepts, prerequisites) |
| `data/items/` _(planned)_ | Quiz/code items with misconception tags and tests |
| `apps/web/` _(planned, after demo)_ | Next.js frontend, Sena UI, Knowledge Graph view |
| `apps/web/messages/` _(planned)_ | i18n strings (`vi` default, `en`) |
| `supabase/migrations/` | Database schema (Postgres + pgvector); `0001_init.sql` = schema v1, `0002` = chunk provenance (chapter, section, kind, url) |
