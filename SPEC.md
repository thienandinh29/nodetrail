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

**Act 2 — breadth (the judge's topic).** At the **start** of the pitch: "Thầy/cô muốn học gì?" → Sena starts
**Deep Research** while we present Act 1 → we return to a finished, cited mini-path; the judge answers 2–3
questions themselves.

Two modes for building a path on any topic:

| | **Nhanh** (quick, NotebookLM-style) | **Nghiên cứu sâu** (Deep Research) |
|---|---|---|
| Sources | Curated, pre-indexed open-textbook library + learner's own upload (stretch) | Agent searches and reads an allowlist of reputable open-education sites live |
| Time | Seconds to ~1 min | Target 3–5 min, hard cap 8 min |
| Cost | Low tokens | High tokens |
| Product role | Free tier; users who won't wait | Quality-first; premium later |
| Demo role | Backup and instant path when the library covers the topic | Act 2 highlight, with a visible research trail |

Quality bar for Deep Research output: ≥ 3 sources from ≥ 2 publishers on the allowlist; prerequisites follow the
sources' own chapter order and learning objectives; every concept and item cited; every item passes an
answerability check against its source; licence recorded per source, no "ND" licences.
Sena shows the trail live ("Đang đọc OpenStax Biology 2e — Chương 8…") and labels output "Bản nháp tự động —
chưa được giáo viên duyệt". MCQ only for non-Python topics.

Safety nets: Deep Research over its cap → Nhanh path; Nhanh can't cover it → pre-generated topics (said openly).
Go/no-go at H12 (see `TEAM.md`).

## Non-goals (for now)
- Gamified streaks/speed rewards.
- Generating curriculum content without a trusted source.
