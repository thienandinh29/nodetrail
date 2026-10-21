# PLAN — build order

No deadline: each phase should be finished properly (tests, docs) before the next.
Update the status column as work progresses.

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

## Open decisions
- Embedding model with good Vietnamese support.
- First pilot subject (suggestion: Python — gradable by test cases).
- Textbook sourcing & licensing.
