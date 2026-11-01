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

## Competition demo (24-hour team build — see `TEAM.md`)
No subject is prepared in advance. A judge names **any** topic, or uploads their own file, and NodeTrail builds a
cited learning path live.

1. **Opening:** "Thầy/cô muốn học gì?" → Sena starts Deep Research (Nhanh gives a first path immediately if the
   library covers the topic, or from the uploaded file).
2. **While it researches (~3–5 min):** the team pitches the problem and solution while the screen shows Sena's
   research trail ("Đang đọc OpenStax Biology 2e — Chương 8…").
3. **The judge learns (~3 min):** graph-walk placement (~6 questions) lights up the map and locates the gap →
   wrong answer → remediation to the weak prerequisite → Socratic hint in Vietnamese with citations →
   correct answer → mastery rises, Sena praises, a review is scheduled.
4. **Dashboard:** time, cost and dropped questions for that topic (stability, real operations).
5. **Fallbacks:** Deep Research over 8 min → Nhanh for the same topic → pre-generated topic (said openly) → backup video.

Two modes and two tiers:

| | **Nhanh** (NotebookLM-style) | **Nghiên cứu sâu** (Deep Research) |
|---|---|---|
| Sources | Curated, pre-indexed open-textbook library + learner's upload (PDF, DOCX, link) | Agent searches and reads an allowlist of reputable open-education sites |
| Time | < 60 s | target 3–5 min, hard cap 8 min |
| Tier / models | Free — cheap models (e.g. Gemini Flash on Vertex AI), no web search | Pro — strong models (e.g. Claude on Vertex AI/Bedrock) + web search |

Deep Research quality bar: ≥ 3 sources from ≥ 2 publishers; prerequisites follow the sources' chapter order and
learning objectives; every concept and item cited; every item passes an answerability check; licence recorded per
source, no "ND" licences. All generated content is labelled "Bản nháp tự động — chưa được giáo viên duyệt".

Demo scope cuts: MCQ only (no code grading), one demo learner, no login. Kept: uploads, both modes, Free/Pro switch,
observability dashboard. Organizer expectations (Spec Kit, architecture, demo, novelty, post-contest path) are
answered in the team doc "NodeTrail — 5 định hướng theo yêu cầu Ban Tổ chức".

## Non-goals (for now)
- Gamified streaks/speed rewards.
- Generating curriculum content without a trusted source.
