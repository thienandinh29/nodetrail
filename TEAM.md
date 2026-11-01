# TEAM — 24-hour parallel build (5 people + AI agents)

Goal: the demo in `SPEC.md` ("Competition demo"): a judge names **any** topic (or uploads a file) and NodeTrail
builds a cited learning path live. No subject is prepared in advance. Every agent reads this file; follow it exactly.

Before H0, ask the organizers: (1) are AI coding agents allowed, (2) may the textbook library be indexed before
the contest, (3) may code written before the contest be reused, (4) how long is the final-round demo (the last pitch
was cut to 3 minutes). If (3) is "no", start a fresh repo with this same plan; (4) decides the demo script.

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
| **P1** (lead) | Platform, infra, integration (2 agents: infra + integration) | `apps/api/app/main.py`, `app/schemas.py`, `app/api/__init__.py`, `app/config.py`, `app/db.py`, `app/jobs/`, `app/observability/`, `supabase/migrations/`, `apps/api/tests/conftest.py`, `scripts/seed.py`, `scripts/pregenerate.py`, `docker/`, `.github/`, `pyproject.toml`/`uv.lock`, `CLAUDE.md`, `TEAM.md`, `PLAN.md`, `MAP.md`, `specs/` setup | Supabase, contracts + API stubs, `topic_jobs` queue + worker, structlog + Langfuse, Google Cloud GPU VM + AWS backup VM, Docker + named Cloudflare Tunnel + GHCR/Watchtower deploy, budget alerts, merges, pre-generated topics, end-to-end runs, demo script |
| **P2** | Sources, research, retrieval (2 agents: research + retrieval) | `app/rag/`, `app/llm/`, `app/research/`, `app/uploads/`, `data/sources/`, `scripts/build_library.*`, `tests/rag/`, `tests/research/`, `tests/uploads/`, `data/eval/` | Allowlist, library of 8–10 open textbooks (~30k chunks, Nhanh mode), Deep Research agent, uploaded-file reader (PDF/DOCX/link), bge-m3 embeddings, cited retrieval, LLM gateway (Vertex AI → Bedrock → OpenRouter) with per-tier routing |
| **P3** | Content generation | `app/content/`, `data/kg/`, `data/items/`, `tests/content/` | Pipeline for any subject: sources → Knowledge Graph → MCQs with a misconception per wrong option → answerability check; validators (no cycles, everything cited) |
| **P4** | Learning flow + tutor | `app/engine/` (new files), `app/graders/`, `app/services/`, `app/tutor/`, `app/api/learn.py`, `app/api/tutor.py`, `tests/engine/`, `tests/graders/`, `tests/services/`, `tests/tutor/` | MCQ grading, **graph-walk placement**, next-item selection, root-cause remediation, Socratic tutor (Vietnamese, cited, no final answers), mastery + review persistence, exam plan |
| **P5** | Web app | `apps/web/`, `assets/sena/` | Next.js app on Vercel: topic input + file upload, research-trail screen, knowledge map (React Flow), learning + tutor screens, Free/Pro switch, Vietnamese i18n, **backup video** |

## Stack (decided; changes go through P1)
- **Web:** Next.js (App Router, TypeScript strict), Tailwind + shadcn/ui, React Flow with **`dagre` auto-layout
  from H0** (never hand-computed x/y; every topic produces a differently shaped graph); deployed on **Vercel**.
- **Cloud (organizer credits):** **Google Cloud is primary, AWS is backup**; swap the roles if AWS approves GPU quota
  first. Region: Singapore (`asia-southeast1` / `ap-southeast-1`), next to Supabase. Budget alerts on both accounts.
- **API host:** Python 3.12, FastAPI, Pydantic v2, `uv`, in Docker on a **GPU VM** on the primary cloud (spec below).
  The backup cloud keeps a **stopped VM with the same image and compose file**, started only if the primary fails.
  Exposed with a **named Cloudflare Tunnel on the team's own domain** (never `trycloudflare.com`: its URL changes on
  every restart and the web app would point at a dead address); `cloudflared` runs with `--protocol http2`. **The VM
  opens no ports to the internet**; all traffic goes through the tunnel. All containers use `restart: unless-stopped`.
  CORS: the Vercel domain and `http://localhost:3000` from H0; shared API key between web and API.
- **Deploy:** Vercel deploys the web app on merge to `main`. For the API, GitHub Actions builds the image and pushes
  it to GHCR; **Watchtower** on the VM pulls new images automatically. Manual fallback on the VM:
  `docker compose pull && docker compose up -d`. P1 keeps a VM setup script (Docker, NVIDIA Container Toolkit,
  GHCR login, cloudflared, Watchtower) so the backup VM can be rebuilt in minutes.
- **Image:** multi-stage build with `uv`: dependencies in a stable layer, `src/` copied last, so a normal merge pulls
  only a few MB. **bge-m3 weights are never in the image**: they live on the VM disk and are bind-mounted
  (`/models/bge-m3:/app/models/bge-m3`). A CPU-only VM uses an ONNX Runtime image (no PyTorch); the GPU VM uses a
  separate GPU image.
- **Jobs (24 h):** a `topic_jobs` table in Postgres is the queue. The worker is a **separate compose service** from the
  same image (`command: python -m app.worker`), never inside uvicorn; it claims jobs with
  `SELECT … FOR UPDATE SKIP LOCKED`. Deep Research runs 3–8 min, never inside a request. Redis + `arq` after the contest.
- **Progress (SSE):** `GET /topics/{id}/events` sends a heartbeat every 15–20 s (Cloudflare drops idle connections
  after ~100 s), with `Cache-Control: no-cache`. The web app reconnects automatically and reloads state from
  `GET /topics/{id}` (backed by `topic_jobs`).
- **Data:** Supabase cloud — Postgres + pgvector (`halfvec`), Storage (uploads). Auth after the contest.
  Free tier is 500 MB: **chunk budget ~30,000 (8–10 textbooks)**, chosen to match the pre-generated topics; check
  the database size after each book and stop at ~350 MB; build the HNSW index after loading, not row by row.
  Fallback if the library must be bigger: Supabase Pro for one month.
- **Embeddings:** BAAI/bge-m3 (1024 dims). Bulk indexing on a **short-lived GPU VM** (delete it right after; Colab as
  fallback); query embeddings on the API VM (GPU, or ONNX int8 if no GPU quota). Hugging Face hosted bge-m3 as backup. **Only the API process holds the
  model**; the worker calls the API's internal `POST /internal/embed` on localhost instead of loading a second copy.
  Encoding runs via `await asyncio.to_thread(...)` so it never blocks the event loop.
- **LLM:** one gateway, `app/llm/`, with provider adapters: **Vertex AI (primary, on credits)**, **Bedrock**, and
  **OpenRouter** (last fallback). `app/llm/routing.yaml` maps (tier, task) → an ordered list of `provider:model`; the
  gateway tries them in order on errors or rate limits. **free** = cheap models (e.g. Gemini Flash on Vertex), Nhanh mode
  only, no web search; **pro** = strong models (e.g. Claude on Vertex or Bedrock) + Deep Research. Exact model IDs chosen
  after the H8 Vietnamese quality test. On OpenRouter avoid `:free` model variants (rate limits).
- **Observability (24 h):** `structlog` JSON logs (`request_id`, `topic_id`, `tier`) + Langfuse for every LLM call
  (model, latency, cost); the Langfuse dashboard is shown at the end of the demo. After the contest:
  OpenTelemetry → Grafana Cloud, Sentry.
- **Engine:** BKT (mastery), `fsrs` (reviews), graph rules (placement, remediation). Pure functions, tested.
- **Tests without Docker:** `tests/conftest.py` creates a **throwaway schema per run** (create → migrate → drop) instead
  of a throwaway database, so every member runs the full suite against **their own free Supabase dev project**. No
  Docker and no bge-m3 on laptops (tests use the fake embedder). CI keeps its own Postgres service. P1, H0–H3.
- **CI:** pytest + ruff required; mypy runs but only warns. Playwright end-to-end tests after the contest;
  during the contest P5 keeps a manual end-to-end checklist.
- **Typed web client:** P5 generates TypeScript types from FastAPI's OpenAPI with `openapi-typescript`, so a contract
  change breaks the web build instead of the demo.

### API VM spec
| | With GPU (recommended) | CPU only (no GPU quota yet) |
|---|---|---|
| Machine | Google Cloud `g2-standard-4` (1× L4) or `n1-standard-4` + 1× T4; AWS `g4dn.xlarge` (1× T4) | 8 vCPU, 32 GB RAM |
| RAM | 16 GB | 32 GB |
| Disk | 50 GB SSD (images + bge-m3 weights) | 50 GB SSD |
| OS | Ubuntu 22.04/24.04 + NVIDIA driver + NVIDIA Container Toolkit | Ubuntu 22.04/24.04 |
| bge-m3 | full model on GPU | ONNX int8 |
| Region | Singapore | Singapore |

Acceptance at H3: query embedding < ~1 s per Vietnamese question on the VM. Bulk-indexing GPU VM is deleted after use;
the backup VM stays stopped until needed.

### Before H0 checklist
- Lead, **today**: request GPU quota on **both** clouds (Singapore + one nearby region; new accounts often start at
  zero and approval can take a day or more); enable Vertex AI and Bedrock and request model access (Claude on both,
  Gemini on Vertex); set budget alerts on both; ask the organizers for the credit amount, end date, whether GPUs and
  model access are included, and whether a card is required.
- Lead: Cloudflare account + a cheap domain + a named tunnel on it; Vercel account linked to the repo; Supabase demo
  project; OpenRouter key with a spending limit (fallback only); members added as GitHub collaborators; cloud
  service-account credentials shared only through a password manager.
- Everyone: Git, Python 3.12, `uv`, their AI coding agent; own Supabase dev project (for running tests) and keys in
  `.env`. Laptops need no GPU and no Docker; 8 GB RAM is enough, 16 GB is comfortable; keep a few GB of disk free.
- P5: Node.js 20 + pnpm. P1: VM setup script tested (`docker run hello-world` works on the VM).

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
| `GET /topics/{id}` | P1 | → `{status: queued\|researching\|indexing\|graph\|items\|ready\|failed, step_vi, progress, subject_slug?, graph_ready, items_ready, sources: [{title, publisher, url, license}], trail: [{at, kind: search\|read\|select, text_vi, url?}]}` (current state from `topic_jobs`; used to recover after a dropped stream) |
| `GET /topics/{id}/events` | P1 | SSE stream of the same fields as they change, plus a `heartbeat` event every 15–20 s |
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
| **H1–H3** | Contracts + API stubs, primary GPU VM (Docker + named tunnel + GHCR/Watchtower), backup VM image, tests switched to a throwaway schema per run | Index 2–3 textbooks on a temporary GPU VM; **measure bge-m3 on the API VM**; first Vertex AI call | Generate a graph from 1 sample chapter | FSRS wrapper + tests; placement design | Wire skeleton to API stubs |
| **H3** checkpoint | Contracts frozen, foundation merged | | | | |
| **H3–H8** | structlog + Langfuse; `topic_jobs` queue + worker + SSE with heartbeat | Retrieval + gateway + Nhanh mode; uploads | Pipeline graph → items → check | Grading, graph-walk placement, next item | Topic input + upload, map, learning screen |
| **H8** checkpoint | Nhanh end to end on **3 unfamiliar topics**. Not solid → whole team on Nhanh + pre-generated topics; Deep Research becomes a bonus. Pick Free/Pro models from the Vietnamese test. | | | | |
| **H8–H11** | Integration, dashboard | Deep Research agent, full library | Quality across subject types | Remediation + Socratic tutor | Research-trail + tutor screens, Free/Pro switch |
| **H11–H12** | Everyone plays judge with an unfamiliar topic; measure time + cost per mode | | | | |
| **H12** | Decide: Deep Research live on stage, or shown as a recording | | | | |
| **H12–H18** | Pre-generate topics, demo script | Fixes, eval | Fixes per subject type | Reviews + exam plan; tutor eval ≥ 18/20 | Polish, manual E2E checklist |
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
| Tunnel URL changes or SSE stream drops mid-research | Named tunnel on own domain; heartbeat every 15–20 s; web reconnects and reloads `GET /topics/{id}` | P1, P5 |
| Primary VM down | Start the backup VM on the other cloud (same image + compose + tunnel token); `/health` check; Render/Railway last resort | P1 |
| GPU quota not approved before H0 | CPU VM (8 vCPU / 32 GB) with ONNX int8; swap primary cloud if the other approves first | P1, P2 |
| Cloud credits burn (GPU VM left running) | Budget alerts on both clouds; delete the indexing VM right after use; backup VM stays stopped | P1 |
| Vertex model access missing or rate-limited | Gateway falls through to Bedrock, then OpenRouter (`routing.yaml` order) | P2 |
| bge-m3 too slow on the host | ONNX int8 → HF hosted → multilingual embedding API + re-index | P2 |
| Docker image too heavy (10–20 min per update) | Weights bind-mounted from the host; multi-stage `uv` build; ONNX image on CPU hosts | P1 |
| Network blocks QUIC/UDP, tunnel won't connect | `cloudflared --protocol http2`; `restart: unless-stopped` | P1 |
| bge-m3 loaded twice (API + worker), host runs out of RAM | Only the API holds the model; worker uses `/internal/embed`; measure RAM at H3 | P1, P2 |
| Database exceeds Supabase free tier | `halfvec`, ~30k-chunk budget, size check per book, HNSW after load; Supabase Pro for a month | P2 |
| Knowledge-map nodes overlap | `dagre`/`elkjs` auto-layout from H0; test shallow/deep and small/large graphs | P5 |
| Deep Research slow, wrong or off-allowlist | 8-min cap → Nhanh → pre-generated topic; domain filter in code | P1, P2, P3 |
| Judge picks a topic with no open textbook | Wikibooks/Wikipedia labelled as encyclopedia; state the limit openly | P2 |
| Generated question wrong in front of judges | Answerability check, "Bản nháp tự động" label, rehearse many subject types | P3 |
| Cheap model returns bad JSON or weak Vietnamese | Pydantic validation + retry, fallback model list, H8 Vietnamese test | P2 |
| Engine/UI only works for one graph shape | Test shallow/deep, small/large graphs; vary topics at every checkpoint | P4, P5 |
| Uploaded file is scanned, table-heavy or has broken fonts | Check text on upload, ask for another file or link, page cap, rehearse with 5–10 real files | P2 |
| LLM cost too high | Measure at H11; lower effort, fewer sources (lead decides); budget alerts on both clouds, spending limit on OpenRouter | P1, P2 |
| Web not ready at H18 | Scripted happy path only; backup video | P5 |
| Venue wifi fails | Pre-generated topics + backup video | P1 |

## Demo acceptance (run at every checkpoint after H8)
1. A teammate plays judge with an unseen topic, or uploads a file → research trail shows allowlisted sources. *(P2, P5)*
2. Path ready (Nhanh < 60 s; Deep Research < 8 min) with a cited knowledge map. *(P2, P3, P5)*
3. Graph-walk placement in ~6 questions locates a gap; the map lights up. *(P4, P5)*
4. Wrong answer → remediation to the weak prerequisite; hint asked → Socratic hint in Vietnamese with citations, no final answer. *(P4)*
5. Correct answer → mastery rises, Sena praises understanding, a review is scheduled. *(P4, P5)*
6. Dashboard shows time, cost and dropped questions for that topic. *(P1)*
