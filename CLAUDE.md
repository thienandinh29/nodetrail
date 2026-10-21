@MAP.md
@SPEC.md
@PLAN.md

# NodeTrail

AI tutor for any subject. The user states a goal ("Tôi muốn học Python") or uploads a syllabus;
NodeTrail builds a Knowledge Graph from trusted textbooks and runs the loop
assess → learn → practice → adjust → review. Mascot: **Sena** the monkey.
Full brief: `context/product-brief.vi.md`.

## Owner & working agreement
- Solo developer (student, aspiring ML engineer) building everything: backend, frontend, ML, infra.
- Big long-running project, no deadline. Build things properly; don't cut corners to "ship an MVP".
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
- **Learning Engine** decides what to learn next: Bayesian Knowledge Tracing, rule engine, FSRS, Elo.
- **Graders** score work objectively per subject: test cases, answer keys, rubrics.
- **LLM + RAG** only explains, always with citations to source textbooks. No citation → don't present it as fact.
- Socratic tutor never hands over the final answer; it gives step-by-step hints.
- Sena praises only real understanding (mastery), never speed or streaks.

## Stack (defaults chosen by Claude, owner approved "your call")
- Monorepo: `apps/web` (Next.js App Router, TypeScript strict, Tailwind + shadcn/ui, pnpm),
  `apps/api` (Python 3.12, FastAPI, Pydantic v2, uv), `packages/` for shared code if needed.
- Supabase: Postgres + pgvector + Auth + Storage. Migrations in `supabase/migrations/`.
- LLM access through one provider-agnostic interface in the API (default: Claude). Never call SDKs directly from features.

## Code conventions
- Python: full type hints, `ruff` (lint + format), `mypy --strict`, `pytest`. Learning-engine math must have unit tests.
- TypeScript: `strict: true`, no `any`, ESLint + Prettier, Vitest for logic.
- Small modules, pure functions for algorithms (BKT/FSRS/Elo) so they are testable without DB or LLM.
- Secrets only in `.env` (never commit, never paste in chat). Document every new variable in `.env.example`.

## Folders
- `context/` — reference material for Claude (brief, research, textbook notes). Read when relevant.
- `outputs/` — generated deliverables (reports, exports, diagrams). Not source code.
