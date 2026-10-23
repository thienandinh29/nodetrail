# PLAN — build order

## Current: competition demo sprint (14 days, full-time)
Scope: see "Competition demo scope" in `SPEC.md`. Update status daily.

| Day | Date | Work | Done when | Status |
|---|---|---|---|---|
| 1 | 10-22 | Scaffolding: uv, FastAPI, Supabase project, schema v1, `/verify` green | pytest passes, tables exist | done |
| 2 | 10-23 | Engine: BKT + `py-fsrs` wrapper, unit-tested | Mastery updates correctly on right/wrong | done |
| 3 | 10-24 | Ingestion: textbook → chunks → embeddings → pgvector | Each chunk keeps chapter/page | done |
| 4 | 10-25 | RAG + LLM gateway, cited answers, 20-question eval | Answers cite the right source | todo |
| 5–6 | 10-26–27 | KG: LLM extracts concepts/prereqs → YAML → owner review → DB | ~40 concepts, no prerequisite loops | todo |
| 7 | 10-28 | **Buffer + checkpoint** | Days 1–6 green on `/verify` | todo |
| 8 | 10-29 | Item bank: misconception-tagged MCQs + code items with tests; grader | ~100 items linked to concepts | todo |
| 9 | 10-30 | Placement test + adaptive item selection | New learner gets a mastery map in ~10 min | todo |
| 10 | 10-31 | Root-cause remediation rules | Scripted scenario remediates correctly | todo |
| 11 | 11-01 | Socratic tutor: hint ladder, citations, no final answers | Answer-extraction tests fail to get the answer | todo |
| 12 | 11-02 | Streamlit UI: goal → graph → quiz → tutor, Vietnamese, Sena | Full loop clickable | todo |
| 13 | 11-03 | FSRS review cards + exam backward schedule + fixes | "Today's review" visible | todo |
| 14 | 11-04 | E2E test, demo script, README, **backup demo video** | Demo runs 3× clean | todo |

Risks / fallbacks:
- KG extraction quality poor → hand-write the graph from the textbook table of contents (~3 h).
- More than 1 day behind at the Day 7 checkpoint → drop Day 13 scope first.
- Live demo failure → have the recorded video and a pre-seeded demo learner ready.

## Long-term (after the demo)
Each phase should be finished properly (tests, docs) before the next. Estimates assume ~20 h/week solo.

| # | Phase | Key deliverables | Status |
|---|---|---|---|
| 0 | Harness | CLAUDE.md, MAP, SPEC, PLAN, git, env | done |
| 1 | Scaffolding | Monorepo, `apps/api` (FastAPI + uv), `apps/web` (Next.js), Supabase local, CI lint/test | todo |
| 2 | Data model | Schema: users, subjects, sources, chunks, concepts, edges, items, misconceptions, attempts, mastery, cards | todo |
| 3 | Learning Engine | BKT, Elo, FSRS, rule engine as pure tested modules | todo |
| 4 | Ingestion + RAG | Textbook ingest → chunks → embeddings (pgvector) → cited retrieval; LLM gateway | todo |
| 5 | Knowledge Graph builder | Extract concepts + prerequisites from sources; graph review tools | todo |
| 6 | Assessment | Placement test, adaptive quiz, misconception-tagged distractors, graders | todo |
| 7 | Adaptive path | Root-cause remediation over the graph | todo |
| 8 | Socratic tutor | Hint ladder, citations, no direct answers | todo |
| 9 | Frontend | Sena UI, graph view, quiz, tutor chat, vi/en i18n | todo |
| 10 | Notes + review | Block notes → cards, FSRS review, exam backward scheduler | todo |
| 11 | Class view | Lecturer dashboards, class weak spots | todo |

## Decisions made
- Pilot subject: Python. Textbook: Think Python 3e, pinned upstream commit (see `sources.version`).
- Embeddings: BAAI/bge-m3, local, 1024 dims (cross-lingual vi ↔ en, works offline).

## Open decisions
- Licensing beyond the demo: Think Python text is CC BY-NC-SA 4.0 — fine for a non-commercial demo,
  but any commercial NodeTrail needs another source or permission, and material derived from the text
  (e.g. generated quiz items) must keep attribution and the same license.
