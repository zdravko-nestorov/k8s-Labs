---
name: ckad-labs
description: Answers Kubernetes and CKAD lab tasks from the user's own lab notes in this workspace, and saves new course labs as numbered Markdown files. Use when the user asks how to do a Kubernetes task, pastes a lab or exam-style task, asks "how did we do this in the labs", or asks to store a new lab. Triggers on kubectl, Pods, Deployments, Jobs, probes, ConfigMaps, Secrets, volumes, ServiceAccounts, and CKAD.
---

# CKAD Labs

The user's lab notes in this workspace are the main knowledge base. Lead with them.

## Knowledge root

- Labs: `labs/NN-*.md` (course labs with worked instructions, `00-` is the first).
- Exams: `exams/NN-*.md` (CKAD practice exams: task, my solution, official solution).
- Challenges: `challenges/NN-*.md` (lab challenges with no official solution: task, my solution, review against the notes).
- Cheat sheet: `exams/CHEATSHEET.md` (fastest command per task type, across all exams and challenges).
- Index: `CATALOG.md` in the workspace root.

Exam files carry the official answers, so they are the best source for fast imperative commands. Their solutions sit in collapsed `<details>` blocks, which grep still reads.

Challenge files carry no official answer. Their review blocks are judgments made from `labs/` and `exams/`, so treat them as second-hand. When a challenge and an exam disagree, the exam wins.

## Two modes

| Mode | Trigger | Action |
|------|---------|--------|
| **Solve** (default) | A Kubernetes task, question, or lab to do | Answer from the notes with exact commands |
| **Ingest** | The user says to create or save a lab, exam, or challenge file | Write a new numbered file and update `CATALOG.md` |

Practice tasks the user solves in chat are **not** saved. Create a file only when the user asks.

## Solve mode

### Find the knowledge

1. Read `CATALOG.md` to pick candidate files by topic. It has a labs table, an exams table, a challenges table, and a topic index.
2. Grep `labs/`, `exams/`, and `challenges/` for task keywords (for example `probe`, `PVC`, `emptyDir`, `securityContext`).
3. Read only the matching files, usually 1 to 3. Stop when they cover the task.

Never read everything into context. If `CATALOG.md` looks stale, glob `labs/*.md`, `exams/*.md`, and `challenges/*.md` instead.

For exam-style or timed tasks, check `exams/CHEATSHEET.md` first, then the exam files. Those hold the official one-line answers.

### Answer

- Lead with the way the notes do it. Match their commands, flags, and field names.
- Add imperative `kubectl` when it is faster, and `--dry-run=client -o yaml` when fields need editing.
- Give YAML when the task needs fields you cannot set imperatively.
- Name the files used, for example `labs/06-probes-and-monitoring.md`.
- If the notes do not cover it, answer from general Kubernetes knowledge and say plainly: not in your notes yet.
- Keep explanations short. One or two sentences per command block.

### Answer template

```markdown
<one line: what this does>

<commands or YAML>

<short why, plus gotchas from the notes>

From: labs/NN-file.md (Step N)
```

For an exam or challenge file, cite the task: `From: exams/01-core-concepts.md (Task 4)`.

## Ingest mode

Run this only when the user explicitly asks to save or create a file. Pasted lab text alone is not a request to save it.

Read `ingest.md` in this skill folder and follow its checklist. That file owns the numbering rule, both file templates, and every place that must be updated.

## Conventions

- Three folders: `labs/` for course labs, `exams/` for practice exams, `challenges/` for lab challenges with no answer key. No deeper nesting.
- Each folder has its own number series. Numbers follow the order the user pastes content, not the course portal order.
- Labs are organised by step. Exams and challenges are organised by task. An exam puts my solution next to the official one. A challenge has no official one, so it puts my solution next to a review written from the notes.
- `Topics:` drives search. Use existing tag names when they fit.
- Update this skill only when a convention changes. New files change `CATALOG.md`, not this file.

## Writing rules

Follow the user's writing style: plain English, short sentences, no em dashes or en dashes, exact commands and version numbers.

## Non-goals

- Saving practice tasks the user solved in chat
- Hint-only coaching, quizzes, or flashcards
- Rewriting existing lab files unless the user asks
