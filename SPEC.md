# SPEC — NodeTrail

Positioning: *An AI tutor for any subject: knows where you are, and leads you onward.*
Target users: Vietnamese self-learners and university students; later lecturers/centers (class view).

## Problem
Median online-course completion is ~12.6% (Jordan, 2015). Self-learners don't know where to start,
forget what they learned, and chatbots answer one question at a time without knowing what to learn next.

## Core loop
Goal or syllabus upload → find trusted textbooks → build Knowledge Graph → **assess → learn → practice → adjust → review**.

## Features
1. **Knowledge Graph** — concepts + prerequisite order extracted from the textbooks. Node brightness = mastery.
2. **Adaptive placement test** — ~10 minutes to estimate the learner's level per concept.
3. **Adaptive Quiz** — items from textbook exercises; every wrong option maps to a specific misconception.
4. **Self-adjusting path** — on errors, trace the root cause back through prerequisites and remediate weak foundations.
5. **Socratic tutor** — never gives the final answer; step-by-step hints; always cites the textbook.
6. **Smart notes** — block-based notes that auto-convert into review cards.
7. **Exam prep** — review schedule planned backwards from the exam date, ~20 minutes/day.

## Engine (deterministic, testable)
- Mastery: Bayesian Knowledge Tracing per learner × concept.
- Item difficulty / learner ability: Elo.
- Spaced repetition: FSRS.
- Path selection & remediation: rule engine over the graph + mastery state.
- Grading: subject-specific objective graders (code test cases, answer keys, rubrics).
- LLM + RAG: explanations only, with citations.

## Non-functional
- Vietnamese-first UI and tutor language; English secondary.
- Every explanation traceable to a source chunk.
- Learner data private by default (Supabase RLS).

## Competition demo scope (sprint 2026-10-22 → 2026-11-04)
Show the three differentiators end to end on **one subject (Python)**: Knowledge Graph, root-cause remediation,
cited Socratic hints in Vietnamese.

| In | Simplified | Out (after demo) |
|---|---|---|
| KG for Python (~40 concepts) | LLM-extracted, then hand-reviewed YAML | Automatic textbook search |
| BKT mastery + root-cause remediation | One curated open-licence textbook | Lecturer/class view |
| Adaptive quiz, misconception-tagged distractors | Placement = graph walk, not full CAT | Block-based notes editor |
| Socratic tutor with citations (vi) | FSRS via `py-fsrs` | Auth / multi-user |
| Code grader (test cases) | Exam schedule = simple backward plan | Elo |
| | Streamlit UI, static Sena image + text | Sena animations, Next.js UI |

Demo story, in two acts:

**Act 1 — depth (Python, ~3 min).** Learner says "Tôi muốn học Python" → placement test → graph lights up →
wrong answer on a loop exercise → NodeTrail traces it to a weak prerequisite (e.g. conditions/boolean logic) →
Socratic hints with citations → mastery updates → review card scheduled.

**Act 2 — breadth (the judge's topic, ~2 min).** "Thầy/cô muốn học gì?" → NodeTrail fetches Wikipedia (vi + en),
builds a **10-concept mini-path** (graph + ~15 MCQs, every item cited) live while Sena shows progress →
the judge answers 2–3 questions themselves.
- Target: graph on screen in ≤ 35 s, first question in ≤ 60 s, hard limit 90 s.
- Safety nets: 10–15 likely topics pre-generated with the same pipeline (said openly); past 90 s, offer the
  closest pre-generated topic; output labelled "Bản nháp tự động — chưa được giáo viên duyệt".
- MCQ only (code grading stays Python-only). Sources: Wikipedia, CC BY-SA, attributed.
- Go/no-go at H12 (see `TEAM.md`): if the live pipeline can't meet the limit, Act 2 becomes "choose from the
  pre-generated topics".

## Non-goals (for now)
- Gamified streaks/speed rewards.
- Generating curriculum content without a trusted source.
