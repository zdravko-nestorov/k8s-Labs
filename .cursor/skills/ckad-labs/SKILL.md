---
name: ckad-labs
description: Answers Kubernetes and CKAD lab tasks from the user's own lab notes in this workspace, and saves new course labs as numbered Markdown files. Use when the user asks how to do a Kubernetes task, pastes a lab or exam-style task, asks "how did we do this in the labs", or asks to store a new lab. Triggers on kubectl, Pods, Deployments, Jobs, probes, ConfigMaps, Secrets, volumes, ServiceAccounts, and CKAD.
---

# CKAD Labs

The user's lab notes in this workspace are the main knowledge base. Lead with them.

## Knowledge root

- Lab files: `NN-*.md` in the workspace root (`00-` is the first lab).
- Index: `CATALOG.md` in the workspace root.

## Two modes

| Mode | Trigger | Action |
|------|---------|--------|
| **Solve** (default) | A Kubernetes task, question, or lab to do | Answer from the notes with exact commands |
| **Ingest** | The user says to create or save a lab file | Write a new numbered file and update `CATALOG.md` |

Practice tasks the user solves in chat are **not** saved. Create a file only when the user asks.

## Solve mode

### Find the knowledge

1. Read `CATALOG.md` to pick candidate labs by topic.
2. Grep the lab files for task keywords (for example `probe`, `PVC`, `emptyDir`, `securityContext`).
3. Read only the matching labs, usually 1 to 3 files. Stop when they cover the task.

Never read all labs into context. If `CATALOG.md` looks stale, glob `NN-*.md` instead.

### Answer

- Lead with the way the notes do it. Match their commands, flags, and field names.
- Add imperative `kubectl` when it is faster, and `--dry-run=client -o yaml` when fields need editing.
- Give YAML when the task needs fields you cannot set imperatively.
- Name the lab files used, for example `06-probes-and-monitoring.md`.
- If the notes do not cover it, answer from general Kubernetes knowledge and say plainly: not in your notes yet.
- Keep explanations short. One or two sentences per command block.

### Answer template

```markdown
<one line: what this does>

<commands or YAML>

<short why, plus gotchas from the notes>

From: NN-file.md (Step N)
```

## Ingest mode

Run this only when the user explicitly asks to save or create a lab file. Pasted lab text alone is not a request to save it.

Read `ingest.md` in this skill folder and follow its checklist. That file owns the numbering rule, the file template, and every place that must be updated.

## Conventions

- Flat folder. No subfolders per module.
- Numbers follow the order the user pastes labs, not the course portal order.
- `Topics:` drives search. Use existing tag names when they fit.
- Update this skill only when a convention changes. New labs change `CATALOG.md`, not this file.

## Writing rules

Follow the user's writing style: plain English, short sentences, no em dashes or en dashes, exact commands and version numbers.

## Non-goals

- Saving practice tasks the user solved in chat
- Hint-only coaching, quizzes, or flashcards
- Rewriting existing lab files unless the user asks
