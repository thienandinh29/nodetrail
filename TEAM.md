# TEAM — 24-hour parallel build (5 people + AI agents)

Goal: finish the **demo story** in `SPEC.md` ("Competition demo scope") in 24 hours.
Already on `main`: schema v1 + 0002, BKT + FSRS engine, Think Python ingestion (405 chunks, bge-m3).
Everything below is what's left of sprint days 4–14, split so five agents can work at once.

## Rules that make parallel work possible
1. **Contracts first.** H0–H2 freezes data formats, API shapes and Python interfaces (below).
   After that, every lane builds against the contract, not against another lane's code.
2. **Stay in your lane.** Edit only the paths your lane owns. Need a change elsewhere? Ask the owner.
3. **Fixtures and stubs until integration.** No lane waits for another; you use sample data
   in the contract format until the real thing is merged.
4. **Small PRs, CI green, P1 merges.** Nobody pushes to `main`.
5. **Feature freeze at H18.** After that, only bug fixes, rehearsal and the backup video.

## Lanes

| Person | Lane | Owns (only edits these) | Delivers |
|---|---|---|---|
| **P1** (lead) | Platform & integration | `app/main.py`, `app/schemas.py`, `app/api/__init__.py`, `app/config.py`, `app/db.py`, `supabase/migrations/`, `tests/conftest.py`, `scripts/seed.py`, `pyproject.toml`/`uv.lock`, `CLAUDE.md`, `TEAM.md`, `PLAN.md`, `MAP.md`, CI | Supabase demo DB, contracts + API stubs, test isolation on Supabase, merges, seed, end-to-end checks, demo script |
| **P2** | RAG & tutor | `app/rag/retrieve.py`, `app/rag/embed.py`, `app/llm/`, `app/tutor/`, `app/api/tutor.py`, `tests/rag/`, `tests/tutor/`, `data/eval/` | HF-hosted bge-m3 + local fallback, cited retrieval, LLM gateway, Socratic hint ladder in Vietnamese, `/ask` + `/tutor/hint` |
| **P3** | Content | `data/kg/`, `data/items/`, `app/content/`, `scripts/draft_*.py`, `tests/content/` | ~40-concept Knowledge Graph, misconceptions, ~100 items (MCQ + code with tests), loaders + validators |
| **P4** | Learning flow | `app/engine/` (new files), `app/graders/`, `app/services/`, `app/api/learn.py`, `tests/engine/`, `tests/graders/`, `tests/services/` | Graders, placement test, next-item selection, root-cause remediation, mastery + review persistence, exam plan, `/placement`, `/attempts`, `/graph`, `/reviews` |
| **P5** | Demo UI | `apps/demo/`, `assets/sena/` | Streamlit app (goal → graph → quiz → tutor → review), Vietnamese UI text, Sena, deploy, **backup video** |

## Contracts (frozen at H2; only P1 changes them, announced at a checkpoint)

### Knowledge Graph — `data/kg/python.yaml` (P3 writes, P4 + P5 read)
```yaml
concepts:
  - slug: boolean-expressions          # kebab-case, unique, never renamed after H10
    name_vi: Biểu thức logic
    name_en: Boolean expressions
    description_vi: Biểu thức cho kết quả True hoặc False.
    chapter_no: 5
    sources: ["https://allendowney.github.io/ThinkPython/chap05.html#boolean-expressions"]
edges:                                  # prerequisite -> concept; loader rejects cycles
  - {from: variables, to: boolean-expressions}
```

### Items — `data/items/<concept-slug>.yaml` (P3 writes, P4 reads)
```yaml
concept: boolean-expressions
misconceptions:
  - slug: assign-vs-compare
    description_vi: Nhầm toán tử gán `=` với so sánh `==`.
items:
  - id: be-001
    kind: mcq
    prompt_vi: "Biểu thức nào trả về True khi x bằng 5?"
    difficulty: 0.3
    source: "https://allendowney.github.io/ThinkPython/chap05.html#boolean-expressions"
    options:
      - {label: "x == 5", correct: true}
      - {label: "x = 5", correct: false, misconception: assign-vs-compare}
  - id: be-002
    kind: code
    prompt_vi: "Viết hàm is_even(n) trả về True nếu n chẵn."
    difficulty: 0.5
    starter: "def is_even(n):\n    pass\n"
    tests: "assert is_even(4)\nassert not is_even(7)\n"
    timeout_s: 3
```
Rule: every wrong MCQ option has a `misconception`; every item has a `source` (CC BY-NC-SA attribution).

### API (FastAPI, JSON). P1 creates all routes on H0–H2 as **stubs returning fixture data**, so P5's UI works from H2.
| Method & path | Owner | Request → Response (Pydantic models live in `app/schemas.py`) |
|---|---|---|
| `POST /learners` | P1 | `{display_name}` → `{learner_id}` |
| `GET /graph?learner_id=` | P4 | → `{nodes: [{slug, name_vi, p_known, mastered}], edges: [{from, to}]}` |
| `POST /placement/next` | P4 | `{learner_id}` → `{item \| null, done}` |
| `GET /next-item?learner_id=` | P4 | → `{item, reason_vi}` |
| `POST /attempts` | P4 | `{learner_id, item_id, answer, hints_used}` → `{correct, misconception_vi?, p_known, mastered, remediation?: {concept_slug, reason_vi}}` |
| `POST /tutor/hint` | P2 | `{learner_id, item_id, level, last_answer?}` → `{hint_vi, citations: [{title, section, url}]}` |
| `POST /ask` | P2 | `{question}` → `{answer_vi, citations}` |
| `GET /reviews/due?learner_id=` | P4 | → `{cards: [{concept_slug, due_at}]}` |
| `POST /exam-plan` | P4 | `{learner_id, exam_date}` → `{days: [{date, concept_slugs, minutes}]}` |
| `POST /topics` | P1 (pipeline: P2 + P3) | `{goal_vi}` → `{topic_id, cached: bool}` (Act 2) |
| `GET /topics/{id}` | P1 | → `{status: queued\|sources\|graph\|items\|ready\|failed, step_vi, progress: 0–1, subject_slug?, graph_ready, items_ready: int}` |
| `GET /topics/cached` | P1 | → `{topics: [{subject_slug, name_vi}]}` (pre-generated list for the fallback picker) |

`item` as sent to the UI never contains the answer: `{id, kind, prompt_vi, options?: [{label}], starter?}`.

### Python interfaces
- `app/rag/embed.py::Embedder.embed(texts) -> list[list[float]]` (exists).
- `app/rag/retrieve.py::search(query: str, k: int = 5) -> list[Citation]`, where `Citation` = `{chunk_id, title, section, url, content}`.
- `app/llm/gateway.py::complete(system: str, messages: list[Message]) -> str`. Tests use a fake; nothing else imports an LLM SDK.
- `app/engine/bkt.py`, `app/engine/srs.py` (exist): pure, no DB.

## Act 2: live topic pipeline ("Thầy/cô muốn học gì?")
A judge names any topic; NodeTrail builds a 10-concept mini-path from Wikipedia while Sena shows progress.
Output is a normal subject (KG + items in the DB), so P4's engine and P5's quiz screens work on it unchanged.

| Step | Owner | How | Time budget |
|---|---|---|---|
| 1. Sources | P2 | Wikipedia API, vi + en, top ~5 articles in parallel; keep title + URL for CC BY-SA attribution | ≤ 3 s |
| 2. Index | P2 | Chunk + **HF-hosted** bge-m3 in one batch (~50 chunks); keep the model warm before the demo | ≤ 5 s |
| 3. Graph | P3 | One LLM call → 10 concepts + prerequisite edges, structured output; validator: no cycles, every concept cites a chunk | ≤ 25 s |
| 4. Items | P3 | **One LLM call per concept, all in parallel** (~1–2 MCQs each, misconception per wrong option, cited) | ≤ 20 s wall clock |
| 5. Check | P3 | Parallel answerability check per item against its cited chunk; drop failures | ≤ 10 s |
| **Total** | | Graph shown after step 3; quiz starts as soon as the first concept's items pass step 5 | **graph ≤ 35 s, first question ≤ 60 s, hard limit 90 s** |

LLM rules for the pipeline (P2 owns `app/llm/`, everyone calls it):
- Model `claude-opus-5-5` (`LLM_MODEL`) at **`effort: low`**, structured outputs (`output_config.format` / `messages.parse`), streaming for progress.
- Parallel item calls with the async client; put the shared source text first and cache it (prompt caching), so the ten
  parallel calls reuse one cached prefix instead of re-reading ~20k tokens each.
- If H2 measurements miss the budget: try fast mode (`speed: "fast"`, beta, ~2.5× output speed, 2× price) or a
  smaller model for items. **That model choice is the lead's call**, not an agent's.
- Check the account's rate limits (requests and tokens per minute) before H10; ten parallel calls must fit.
- Set a spending limit in the Anthropic console before rehearsals.

Pre-generation: P1 runs the same pipeline the night before on 10–15 likely topics (lead picks the list).
**Go/no-go at H12:** live run under 90 s on 3 test topics → keep live; otherwise Act 2 = pick from pre-generated topics.

## Timeline (H0 = kickoff)

| Window | P1 lead | P2 RAG & tutor | P3 content | P4 learning flow | P5 demo UI |
|---|---|---|---|---|---|
| **H0–H2** contracts | Supabase demo project; copy 0001+0002 schema and the 405 vectors; write `app/schemas.py` + stub routes; switch tests to a throwaway schema per run | Set up own Supabase dev project; measure HF-hosted bge-m3 (cold, warm, errors over 50 queries) | Set up; list glossary + section chunks per chapter as raw material | Set up; design remediation rules on paper against the KG contract | Set up; Streamlit skeleton calling the stub API |
| **H2–H6** build | Seed script (loads KG + items YAML into DB); review first PRs | `retrieve.search` + HF/local fallback; 20-question vi eval set | LLM-draft concepts + edges from glossaries/TOC → `python.yaml`; loader with cycle check | MCQ grader + code grader (subprocess, timeout, no network); mastery persistence via BKT | Goal screen, graph view (nodes coloured by `p_known`), quiz screen with fixture data |
| **H6** checkpoint | Merge whatever is green; everyone rebases | Draft PR | KG PR (owner review: you sign off the graph) | Graders PR | Draft PR |
| **H6–H10** build | Wire real routes as lanes land; e2e test skeleton | LLM gateway; hint ladder (3 levels, citations, never the final answer); `/ask` | Misconceptions + MCQs for all concepts; code items for ~15 concepts | Placement (walk the graph from roots); next-item selection | Tutor panel, Sena feedback (praise only on mastery), Vietnamese copy |
| **H10–H12** integration 1 | Seed demo DB with real KG + items; first full click-through; log bugs per lane | Fix integration bugs | Fill gaps found in click-through | Fix integration bugs | Switch UI from stubs to real API |
| **H12–H18** complete | Run demo story end to end every 2 h; triage | Answer-extraction tests (tutor must refuse to give answers); eval ≥ 18/20 | Validate: every option tagged, every item sourced; ~100 items | Root-cause remediation (wrong answer → weakest prerequisite); review cards + exam plan | Review screen, exam plan screen, polish; deploy to Streamlit Community Cloud |
| **H12** Act 2 go/no-go | Decide live vs pre-generated (see Act 2) | Live pipeline timing on 3 topics | Pipeline quality on 3 topics | — | Progress screen + fallback picker |
| **H18** freeze | Tag `demo-v1` | — | — | — | — |
| **H18–H23** rehearse | Demo script + pitch; 3 clean runs; pre-seed a demo learner | Warm-up script for HF model | Proof-read Vietnamese text | Scripted remediation scenario test | **Record backup video**; offline fallback check |
| **H23–H24** | Buffer | Buffer | Buffer | Buffer | Buffer |

## Git workflow
- One branch per lane: `p1/platform`, `p2/rag-tutor`, `p3/content`, `p4/learning-flow`, `p5/demo-ui`.
  Running several agents for one lane? Give each its own worktree (`git worktree add ../nt-p4-graders -b p4/graders`).
- **Rebase at every checkpoint**: `git fetch && git rebase origin/main`, then `/verify`.
- PR title `[P3] Knowledge Graph v1`; CI must be green; P1 merges. Stacked PRs: **retarget to `main` before deleting the base branch** (deleting first closes the PR).
- **Migrations**: numbers reserved so nobody collides — `0003` P4 (flow/attempt extras), `0004` P3 (content extras), `0005` P2 (tutor logs). Additive only. P1 merges them in order.
- **Dependencies**: add with `uv add` in your own PR; on `uv.lock` conflicts, take `main`'s lock and re-run `uv lock`.
- **Secrets**: each person makes their own free Supabase dev project and HF token and fills their own `.env`.
  The demo database belongs to P1. Share demo keys only through a password manager, never in chat or git.

## Working with your AI agent
Start every agent session in your lane's worktree. It auto-loads `CLAUDE.md`, which loads this file.
First prompt (copy and fill in):
> You are working on lane **P_ (name)** in `TEAM.md`. Only edit the paths that lane owns.
> Task: **(row from the timeline for the current window)**. Follow the contracts in `TEAM.md` exactly;
> if one seems wrong, stop and tell me instead of changing it. Use fixtures for anything another lane owns.
> When done: run `/verify`, update `MAP.md` lines for new paths, and give me a PR description.

Agent rules: never push to `main`, never edit another lane's files, never change a contract, never read `.env`.

## Definition of done (every PR)
- `/verify` green; new logic has tests; no real LLM or network calls in tests (use fakes).
- User-facing text in Vietnamese; explanations carry citations.
- Matches the contract shapes exactly; `MAP.md` updated for new paths.

## Checkpoints (H6, H10, H12, then every 2 h; 15 minutes)
Each person says **done / next / blocked**. P1 merges green PRs, everyone rebases, and blockers
get an owner and a deadline before the call ends.

## Risks and fallbacks
| Risk | Fallback | Owner |
|---|---|---|
| HF-hosted model slow or flaky | Local bge-m3 (same vectors); last resort Voyage + re-index | P2 |
| LLM-drafted graph is poor | Hand-write ~40 concepts from the book's table of contents (~2 h) | P3 |
| Code grader hangs or is unsafe | Timeout + no network; demo runs locally only | P4 |
| UI not ready at H18 | Demo through the scripted happy path only; backup video | P5 |
| Venue wifi fails | Local model + local DB snapshot + backup video | P1 |
| Act 2 live generation too slow or wrong | Pre-generated topics + 90 s timeout + "bản nháp tự động" label; H12 go/no-go | P1, P3 |
| LLM rate limit hit by parallel calls | Lower concurrency; pre-warm cache; fall back to pre-generated topic | P2 |

## Demo story acceptance (run at every checkpoint after H10)
1. Learner types "Tôi muốn học Python" → placement starts. *(P5, P4)*
2. ~10 questions → graph lights up by mastery. *(P4, P3, P5)*
3. Wrong answer on a loop exercise → remediation points to a weak prerequisite (e.g. boolean expressions). *(P4, P3)*
4. Learner asks for help → Socratic hints in Vietnamese with book citations, no final answer. *(P2)*
5. Correct answer → mastery rises, Sena praises understanding, a review card is scheduled. *(P4, P5)*
6. **Act 2:** a teammate plays judge with an unseen topic (e.g. "luật bóng đá") → graph ≤ 35 s, first question ≤ 60 s,
   every question cited; past 90 s the fallback picker appears. *(P1, P2, P3, P5)*
