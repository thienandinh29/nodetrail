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

## Non-goals (for now)
- Gamified streaks/speed rewards.
- Generating curriculum content without a trusted source.
