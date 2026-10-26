# TEAM — 24-hour parallel build (5 people + AI agents)

Goal: the demo in `SPEC.md` ("Competition demo"): a judge names **any** topic (or uploads a file) and NodeTrail
builds a cited learning path live. No subject is prepared in advance. Every agent reads this file; follow it exactly.

Before H0, ask the organizers: (1) are AI coding agents allowed, (2) may the textbook library be indexed before
the contest, (3) may code written before the contest be reused. If (3) is "no", start a fresh repo with this same plan.

## Rules that make parallel work possible
1. **Contracts first.** H0–H3 freezes data formats, API shapes and Python interfaces (below).
2. **Stay in your lane.** Edit only the paths your lane owns. Need a change elsewhere? Ask the owner.
3. **Fixtures and stubs until integration.** No lane waits for another.
4. **Small PRs, CI green, P1 merges.** Nobody pushes to `main`.
5. **Spec Kit per feature:** spec → plan → tasks in `specs/<feature>/` before an agent writes code.
6. **Feature freeze at H18.** After that, only bug fixes, rehearsal and the backup video.

## Lanes

| Person | Lane | Owns (only edits these) | Delivers |
|---|---|---|---|
| **P1** (lead) | Platform, infra, integration (2 agents: infra + integration) | `apps/api/app/main.py`, `app/schemas.py`, `app/api/__init__.py`, `app/config.py`, `app/db.py`, `app/jobs/`, `app/observability/`, `supabase/migrations/`, `apps/api/tests/conftest.py`, `scripts/seed.py`, `scripts/pregenerate.py`, `docker/`, `.github/`, `pyproject.toml`/`uv.lock`, `CLAUDE.md`, `TEAM.md`, `PLAN.md`, `MAP.md`, `specs/` setup | Supabase, contracts + API stubs, Redis + worker, logging/tracing/Sentry/Langfuse, Docker + Cloudflare Tunnel deploy, merges, pre-generated topics, end-to-end runs, demo script |
| **P2** | Sources, research, retrieval (2 agents: research + retrieval) | `app/rag/`, `app/llm/`, `app/research/`, `app/uploads/`, `data/sources/`, `scripts/build_library.*`, `tests/rag/`, `tests/research/`, `tests/uploads/`, `data/eval/` | Allowlist, library of 15–20 open textbooks (Nhanh mode), Deep Research agent, uploaded-file reader (PDF/DOCX/link), bge-m3 embeddings, cited retrieval, OpenRouter gateway with per-tier routing |
| **P3** | Content generation | `app/content/`, `data/kg/`, `data/items/`, `tests/content/` | Pipeline for any subject: sources → Knowledge Graph → MCQs with a misconception per wrong option → answerability check; validators (no cycles, everything cited) |
| **P4** | Learning flow + tutor | `app/engine/` (new files), `app/graders/`, `app/services/`, `app/tutor/`, `app/api/learn.py`, `app/api/tutor.py`, `tests/engine/`, `tests/graders/`, `tests/services/`, `tests/tutor/` | MCQ grading, **graph-walk placement**, next-item selection, root-cause remediation, Socratic tutor (Vietnamese, cited, no final answers), mastery + review persistence, exam plan |
| **P5** | Web app | `apps/web/`, `assets/sena/` | Next.js app on Vercel: topic input + file upload, research-trail screen, knowledge map (React Flow), learning + tutor screens, Free/Pro switch, Vietnamese i18n, **backup video** |

## Stack (decided; changes go through P1)
- **Web:** Next.js (App Router, TypeScript strict), Tailwind + shadcn/ui, React Flow; deployed on **Vercel**.
- **API:** Python 3.12, FastAPI, Pydantic v2, `uv`. Runs in Docker on a **teammate's machine**, exposed with
  **Cloudflare Tunnel**; the same image deploys to Render/Railway as fallback. CORS limited to the Vercel domain;
  shared API key between web and API.
- **Jobs:** Redis + `arq` worker on the same host (Deep Research runs 3–8 min, never inside a request);
  progress to the web over Server-Sent Events.
- **Data:** Supabase cloud — Postgres + pgvector (`halfvec`), Storage (uploads), Auth (after the demo).
- **Embeddings:** BAAI/bge-m3 (1024 dims). Bulk indexing on a free Colab/Kaggle GPU; query embeddings on the host
  (GPU if available, ONNX int8 on CPU). Hugging Face hosted bge-m3 as backup.
- **LLM:** OpenRouter only, through `app/llm/`. Model per (tier, task) in `app/llm/routing.yaml`:
  **free** = cheap models, Nhanh mode only, no web search; **pro** = strong models + Deep Research.
  Exact model IDs chosen after the H8 Vietnamese quality test. Avoid `:free` model variants (rate limits).
- **Observability:** `structlog` JSON logs (`request_id`, `topic_id`, `tier`), OpenTelemetry → Grafana Cloud,
  Sentry for errors, Langfuse for every LLM call (model, latency, cost).
- **Engine:** BKT (mastery), `fsrs` (reviews), graph rules (placement, remediation). Pure functions, tested.
- **CI:** pytest + ruff required; mypy runs but only warns. Playwright for the end-to-end flow (P5).

### Host machine requirements (check before H0)
| | Minimum | Recommended |
|---|---|---|
| RAM | 8 GB (ONNX int8 only, nothing else open) | 16 GB+ |
| CPU | 4 cores, recent | 6–8 cores |
| GPU | optional | NVIDIA, 4 GB+ VRAM |
| Free disk | 15 GB | 30 GB |
| Network | stable | wired, ≥ 10 Mbps upload |
| Uptime | on for 24 h + demo day | plugged in; sleep and auto-restart off |

Acceptance at H3: query embedding < ~1 s per Vietnamese question on the host. Otherwise move the API to Render/Railway.

## Contracts (frozen at H3; only P1 changes them, announced at a checkpoint)

### Knowledge Graph (P3 writes, P4 + P5 read) — stored in the DB; YAML for fixtures
```yaml
concepts:
  - slug: photosynthesis-light-reactions   # kebab-case, unique per subject
    name_vi: Pha sáng của quang hợp
    name_en: Light reactions
    description_vi: ...
    sources: ["https://openstax.org/books/biology-2e/pages/..."]
edges:                                      # prerequisite -> concept; loader rejects cycles
  - {from: cell-structure, to: photosynthesis-light-reactions}
```

### Items (P3 writes, P4 reads) — MCQ only
```yaml
concept: photosynthesis-light-reactions
misconceptions:
  - {slug: oxygen-from-co2, description_vi: "Nghĩ rằng O2 sinh ra từ CO2"}
items:
  - id: plr-001
    kind: mcq
    prompt_vi: "Oxy sinh ra trong quang hợp có nguồn gốc từ đâu?"
    difficulty: 0.4
    source: "https://openstax.org/..."
    options:
      - {label: "Từ nước (H2O)", correct: true}
      - {label: "Từ CO2", correct: false, misconception: oxygen-from-co2}
```
Every wrong option has a `misconception`; every item has a `source`; items failing the answerability check are dropped.

### API (FastAPI, JSON; Pydantic models in `app/schemas.py`). P1 ships all routes as **stubs with fixture data** by H3.
| Method & path | Owner | Request → Response |
|---|---|---|
| `POST /learners` | P1 | `{display_name}` → `{learner_id}` |
| `POST /topics` | P1 (pipeline P2 + P3) | `{goal_vi?, upload_id?, mode: "quick"\|"deep", tier: "free"\|"pro"}` → `{topic_id, cached}` |
| `GET /topics/{id}` | P1 | → `{status: queued\|researching\|indexing\|graph\|items\|ready\|failed, step_vi, progress, subject_slug?, graph_ready, items_ready, sources: [{title, publisher, url, license}], trail: [{at, kind: search\|read\|select, text_vi, url?}]}` (also streamed as SSE) |
| `GET /topics/cached` | P1 | → `{topics: [{subject_slug, name_vi}]}` |
| `POST /uploads` | P2 | multipart PDF/DOCX or `{url}` → `{upload_id, pages, text_ok}` |
| `GET /graph?learner_id=&subject=` | P4 | → `{nodes: [{slug, name_vi, p_known, mastered}], edges: [{from, to}]}` |
| `POST /placement/next` | P4 | `{learner_id, subject}` → `{item \| null, done, gap_slug?}` |
| `GET /next-item?learner_id=&subject=` | P4 | → `{item, reason_vi}` |
| `POST /attempts` | P4 | `{learner_id, item_id, answer, hints_used}` → `{correct, misconception_vi?, p_known, mastered, remediation?: {concept_slug, reason_vi}}` |
| `POST /tutor/hint` | P4 | `{learner_id, item_id, level, last_answer?, tier}` → `{hint_vi, citations: [{title, section, url}]}` |
| `POST /ask` | P2 | `{question, subject, tier}` → `{answer_vi, citations}` |
| `GET /reviews/due?learner_id=` | P4 | → `{cards: [{concept_slug, due_at}]}` |
| `POST /exam-plan` | P4 | `{learner_id, subject, exam_date}` → `{days: [{date, concept_slugs, minutes}]}` |

`item` sent to the web never contains the answer: `{id, kind, prompt_vi, options: [{label}]}`.

### Python interfaces
- `app/rag/embed.py::Embedder.embed(texts) -> list[list[float]]` (exists; add ONNX/GPU/hosted implementations).
- `app/rag/retrieve.py::search(query, subject, k=5) -> list[Citation]`.
- `app/llm/gateway.py::complete(task, tier, system, messages, schema=None) -> str | BaseModel` — reads
  `routing.yaml`, validates JSON with Pydantic, retries, falls back to the next model. Nothing else imports an LLM SDK.
- `app/research/agent.py::research(goal, on_event) -> list[ChosenSource]` — allowlisted domains only,
  ≤ 15 searches, ≤ 12 page reads, 8-minute cap.
- `app/engine/bkt.py`, `app/engine/srs.py` (exist): pure, no DB.

### Placement (graph walk, P4)
Start at an advanced concept. Correct → mark its prerequisites as known (raise `p_known`) and move to the next
unknown branch. Wrong → ask about one of its prerequisites. Stop after ~6 questions or when the gap is located;
return `gap_slug`. BKT keeps updating mastery after every later attempt.

## Two modes
| | Nhanh (NotebookLM-style) | Nghiên cứu sâu (Deep Research) |
|---|---|---|
| Sources | Curated library + learner's upload | Agent searches and reads the allowlist live |
| Time | < 60 s | target 3–5 min, cap 8 min |
| Tier | Free (cheap models) | Pro (strong models) |

Allowlist (`data/sources/allowlist.yaml`, P1 approves): OpenStax, LibreTexts, MIT OpenCourseWare, Saylor Academy,
Open Textbook Library, Wikibooks; Wikipedia last resort, labelled. No "ND" licences; licence recorded per source.
Deep Research quality bar: ≥ 3 sources from ≥ 2 publishers; prerequisites from the sources' chapter order and
objectives; everything cited; answerability-checked items.

## Timeline (H0 = kickoff)
| Window | P1 | P2 | P3 | P4 | P5 |
|---|---|---|---|---|---|
| **H0–H1** | Repo, CLAUDE/TEAM, CI, FastAPI skeleton, schema on Supabase, Spec Kit setup | Allowlist + HTML reader (own folder) | Graph/item formats + validators (own folder) | BKT module (own folder) | Next.js skeleton, Sena assets |
| **H1–H3** | Contracts + API stubs, Docker + tunnel on host, Redis | Index 2–3 textbooks on Colab; **measure bge-m3 on the host** | Generate a graph from 1 sample chapter | FSRS wrapper + tests; placement design | Wire skeleton to API stubs |
| **H3** checkpoint | Contracts frozen, foundation merged | | | | |
| **H3–H8** | Logging/tracing/Sentry/Langfuse; job queue + SSE | Retrieval + gateway + Nhanh mode; uploads | Pipeline graph → items → check | Grading, graph-walk placement, next item | Topic input + upload, map, learning screen |
| **H8** checkpoint | Nhanh end to end on **3 unfamiliar topics**. Not solid → whole team on Nhanh + pre-generated topics; Deep Research becomes a bonus. Pick Free/Pro models from the Vietnamese test. | | | | |
| **H8–H11** | Integration, dashboard | Deep Research agent, full library | Quality across subject types | Remediation + Socratic tutor | Research-trail + tutor screens, Free/Pro switch |
| **H11–H12** | Everyone plays judge with an unfamiliar topic; measure time + cost per mode | | | | |
| **H12** | Decide: Deep Research live on stage, or shown as a recording | | | | |
| **H12–H18** | Pre-generate topics, demo script | Fixes, eval | Fixes per subject type | Reviews + exam plan; tutor eval ≥ 18/20 | Polish, Playwright E2E |
| **H18** freeze | Tag `demo-v1` | | | | |
| **H18–H23** | 3 clean runs with unseen topics; pitch timed to the research | | | | **Backup video** |
| **H23–H24** | Buffer | | | | |

## Git workflow
- Branch per lane: `p1/platform`, `p2/sources`, `p3/content`, `p4/learning`, `p5/web`; one worktree per extra agent.
- **Rebase at every checkpoint**: `git fetch && git rebase origin/main`, then run tests.
- PR title `[P3] ...`; CI green; P1 merges. Stacked PRs: **retarget to `main` before deleting the base branch**.
- Migrations reserved: `0003` P4, `0004` P3, `0005` P2 (`halfvec` + uploads), `0006` P1 (`topic_jobs`, `llm_calls`). Additive only.
- Dependencies via `uv add` / `pnpm add` in your own PR; on lockfile conflicts take `main`'s and re-lock.
- **Secrets:** each person has their own Supabase dev project and keys in `.env`. Demo keys only via a password manager.

## Working with your AI agent
First prompt (copy and fill in):
> You are working on lane **P_ (name)** in `TEAM.md`. Only edit the paths that lane owns.
> Task: **(your row in the timeline for the current window)**. Write or update `specs/<feature>/spec.md`,
> `plan.md` and `tasks.md` first. Follow the contracts in `TEAM.md` exactly; if one seems wrong, stop and tell me.
> Use fixtures for anything another lane owns. When done: run tests, update `MAP.md`, write a PR description.

Agent rules: never push to `main`, never edit another lane's files, never change a contract, never read `.env`.

## Definition of done (every PR)
pytest + ruff green; new logic has tests; no real LLM/network calls in tests; user-facing text in Vietnamese;
explanations cited; contract shapes exact; `MAP.md` updated.

## Risks and fallbacks
| Risk | Fallback | Owner |
|---|---|---|
| Repo/contracts late past H3 | Lanes keep working in their own folders; P1 does nothing else in H0–H3 | P1 |
| Host machine off, asleep or offline | Sleep off, wired network, `/health` check; image ready for Render/Railway; API on the demo laptop at the venue | P1 |
| bge-m3 too slow on the host | ONNX int8 → HF hosted → multilingual embedding API + re-index | P2 |
| Deep Research slow, wrong or off-allowlist | 8-min cap → Nhanh → pre-generated topic; domain filter in code | P1, P2, P3 |
| Judge picks a topic with no open textbook | Wikibooks/Wikipedia labelled as encyclopedia; state the limit openly | P2 |
| Generated question wrong in front of judges | Answerability check, "Bản nháp tự động" label, rehearse many subject types | P3 |
| Cheap model returns bad JSON or weak Vietnamese | Pydantic validation + retry, fallback model list, H8 Vietnamese test | P2 |
| Engine/UI only works for one graph shape | Test shallow/deep, small/large graphs; vary topics at every checkpoint | P4, P5 |
| Uploaded file is scanned, table-heavy or has broken fonts | Check text on upload, ask for another file or link, page cap, rehearse with 5–10 real files | P2 |
| LLM cost too high | Measure at H11; lower effort, fewer sources (lead decides); spending limit on OpenRouter | P1, P2 |
| Web not ready at H18 | Scripted happy path only; backup video | P5 |
| Venue wifi fails | Pre-generated topics + backup video | P1 |

## Demo acceptance (run at every checkpoint after H8)
1. A teammate plays judge with an unseen topic, or uploads a file → research trail shows allowlisted sources. *(P2, P5)*
2. Path ready (Nhanh < 60 s; Deep Research < 8 min) with a cited knowledge map. *(P2, P3, P5)*
3. Graph-walk placement in ~6 questions locates a gap; the map lights up. *(P4, P5)*
4. Wrong answer → remediation to the weak prerequisite; hint asked → Socratic hint in Vietnamese with citations, no final answer. *(P4)*
5. Correct answer → mastery rises, Sena praises understanding, a review is scheduled. *(P4, P5)*
6. Dashboard shows time, cost and dropped questions for that topic. *(P1)*
