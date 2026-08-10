# Ingest a new lab or exam

Follow every step. Do not skip the duplicate check or the catalog updates.

Run this only when the user explicitly asks to save or create a file.

## 1. Pick the type

| Type | Looks like | Goes to |
|------|-----------|---------|
| **Lab** | Course lab with worked instructions to follow | `labs/NN-short-kebab-title.md` |
| **Exam** | Practice exam: numbered checks the user solves alone, **plus a solution guide** | `exams/NN-domain.md` |
| **Challenge** | Lab challenge: numbered checks the user solves alone, **no solution guide** | `challenges/NN-short-kebab-title.md` |

If the paste says "Practice Exam", has "validation checks", or shows a score like `0/6`, it is an exam or a challenge. The solution guide is what separates them. No step 3 with official answers means it is a challenge.

Each folder has its own number series. An exam number never blocks a lab number.

## 2. Duplicate check

Grep `labs/` and `exams/` for the title and for one distinctive command from the pasted text.

If a file already covers it, tell the user which file and stop. Do not create a second file.

## 3. Pick the number and name

List the target folder and take the next free number. Zero-pad to two digits, for example `13`.

Numbers follow the order the user pastes content, not the course portal order.

File name rules:

- `NN-short-kebab-title.md`
- Three to six words in the title
- Lowercase, hyphens, no spaces
- Drop filler words from the course title
- For exams, name by domain: `01-core-concepts.md`, `02-configuration.md`

## 4a. Write a lab file

```markdown
# NN - Title

> Course: Cloud Native Champions: CKAD Bootcamp  
> Module: <module name, if given>  
> Lab: <exact lab title>  
> Topics: <comma separated tags>

## Goal
<one or two sentences: what the lab practises>

## Notes
<definitions, gotchas, exam tips, defaults worth remembering>

## Steps
### Step 1 - <name>
<numbered instructions, with fenced bash or yaml blocks>

## Validation
<lab validation checks, if the lab has them>

## Summary
<one or two sentences>
```

- Multi-step labs stay in one file. One `### Step N - <name>` per lab step.
- Move repeated warnings and definitions into `## Notes` so the steps stay short.

## 4b. Write an exam file

Organise by **task**, not by the portal's steps. A practice exam paste usually has three steps: the tasks, the user's solution, and the solution guide. Those are three views of the same tasks. Merge them so each task is readable in one place.

Solutions go in collapsed `<details>` blocks so the file can be re-attempted without spoilers. The requirement and the difference notes stay open.

````markdown
# Exam NN - Domain

> Course: Cloud Native Champions: CKAD Bootcamp  
> Type: CKAD Practice Exam  
> Domain: <domain>  
> Checks: <count>  
> Solution guide: <url, if given>  
> Topics: <comma separated tags>

## Format
<how the exam is scored and what is allowed>

## Fast answers
<one bash block, the shortest correct command per check, each with a # comment>

## Exam strategy notes
<habits that apply across tasks, not per-task detail>

---

## Task N - <task title>
Namespace `<ns>` · Skill: <the one technique this check drills>

**Requirement**

- <what the check asks for, as a list>

<details><summary><b>My solution</b></summary>

<the user's commands, unchanged>

</details>

<details><summary><b>Official solution</b></summary>

<the solution guide commands>

</details>

<details><summary><b>Technique and what to review</b></summary>

- <official commentary, condensed to the reusable technique>

Review: <links to labs/ files that cover this>

Docs: <kubernetes.io links>

</details>

**Difference that matters**

<why one solution is faster or safer, or "none, both are equivalent">

---

## What to take into the next exam
<numbered list of the reusable lessons>
````

Extra rules for exam files:

- Keep the user's solution even when the official one is better. Comparing the two is the point.
- If both solutions are the same, write "none, both are equivalent" and move on.
- Check the official solution before storing it. If it misses a requirement, keep it as given, add one line inside the block pointing down, and explain the gap in the difference notes. Do not silently correct it.
- Condense the official Commentary into the technique block. Keep the reusable habit, drop the restatement of what the command does. No verbatim quoting, the solution guide URL is in the header.
- Turn the official "Content to review" list into links to `labs/` files. Match by topic using the topic index in `CATALOG.md`. If no lab covers it, say so and name the closest ones. Those gaps are worth knowing.
- Turn "Suggested documentation bookmarks" into real kubernetes.io URLs. Drop repeats, the kubectl Cheat Sheet appears in almost every check.
- Blank lines are required after `<summary>` and before `</details>`, otherwise fenced code inside will not render.
- Record fields that have no imperative flag, for example `terminationGracePeriodSeconds`. Those are the ones that force YAML.

## 4c. Write a challenge file

Same task-by-task layout as an exam, with one swap. There is no official solution, so the second
collapsed block is a **review against the notes**: does this match what `labs/` and `exams/` already
say, what is missing, what is risky, what is faster.

The point of a challenge file is the self-assessment, so two extra parts carry it. A score card near
the top gives a verdict per task at a glance. A pattern check at the end names the strengths and the
repeat mistakes across tasks.

````markdown
# Challenge NN - Title

> Course: Cloud Native Champions: CKAD Bootcamp  
> Type: Lab Challenge, no official solution  
> Checks: <count>  
> Result: <passed> of <count> passed  
> Reviewed against: `labs/`, `exams/`, `exams/CHEATSHEET.md`  
> Topics: <comma separated tags>

## Format
<how it is scored, plus a line saying nothing here is an official answer>

## Score card

| Task | Result | What it drills |
|------|--------|----------------|
| N - <short title> | Correct / Correct, not verified / Correct with a caveat / Unfinished / Wrong | <the one technique> |

<one or two sentences naming the pattern across the results>

## Best commands I know now
<one bash block, the shortest correct command per check, from the notes, not from a solution guide>

---

## Task N - <task title>
Namespace `<ns>` · Skill: <the one technique this check drills>

**Requirement**

- <what the check asks for, as a list>

<details><summary><b>My solution</b></summary>

<the user's commands, unchanged>

</details>

<details><summary><b>Review against my notes</b></summary>

**Verdict: <one line>**

- <what is right, and which note it matches>
- <what is missing, wrong, or risky, and why>
- <the faster or safer command>

Source in my notes: <links to labs/ and exams/ files>

Docs: <kubernetes.io links>

</details>

**What to fix next time**

<one or two sentences, or "Nothing. Keep this one as it is.">

---

## Pattern check

**Doing well**
<numbered list>

**To fix**
<numbered list>
````

Extra rules for challenge files:

- Never invent an official answer. Say the verdict comes from the notes, and name the file it comes from.
- A verdict must be earned. Link the lab or exam that backs it, or verify the command and say so.
- Ask which checks the grader passed before writing the score card. A `???` or a missing note means the user was unsure, not that the check failed. Grade the commands, and record the pass separately.
- Keep a risk note even when a command passed. Say why it worked this time and when the same command would fail.
- If a task is unfinished, that is the most valuable entry in the file. Give the ordered diagnostic, list every fault that fits the symptom, and say plainly which parts cannot be settled without the cluster.
- Praise is only useful when it is specific. "Used the cheat sheet entry from exam 05 task 1 without detouring through YAML" beats "good".
- Check whether a mistake already appears in the notes. A repeat mistake is the finding worth recording.
- Same rendering rules as exam files: blank lines after `<summary>` and before `</details>`.

## 4d. Update the exam cheat sheet

After an exam or challenge file, fold its fastest commands into `exams/CHEATSHEET.md`.

- Group by task type, not by exam.
- Tag each entry with its source, `[E02 t3]` for an exam, `[C01 t1]` for a challenge.
- A challenge entry has no official backing. Only add one when a lab, an exam, or a verified command supports it.
- Replace an existing entry when the new exam shows a faster way. Keep one command per job.
- Add any new no-flag field to the table there.
- Add the exam or challenge to the "Exams covered" table.
- New section? Two edits go with it. Add a row to the Contents table at the top, in the same order
  as the sections, and put `<sup>[↑ contents](#contents)</sup>` on the line directly below the new
  `##` heading, with no blank line between. Every section except Contents has one.
- Never put the arrow inside the heading text. Anchors are built from the heading, lowercased,
  spaces to hyphens, other punctuation dropped, so `## Pods ↑` becomes `#pods-` and the Contents
  row breaks. `## Jobs and CronJobs` is `#jobs-and-cronjobs`. Renaming a heading breaks its row too.

## 5. Rules for all content

- Keep the user's commands, flags, and wording. Fix only structure, formatting, and clear typos.
- Put every command in a fenced block: `bash` for shell, `yaml` for manifests.
- Keep inline notes that explain placeholders, for example `...` replaced by tab completion.
- Never store real credentials. Replace a password or key with a pointer to the lab UI.
- Reuse existing tag names in `Topics:` where they fit. Check the topic index in `CATALOG.md`.

## 6. Update the table in CATALOG.md

Add a row in number order, in the labs table, the exams table, or the challenges table.

```markdown
| NN | Lab title | [labs/NN-file.md](labs/NN-file.md) | <step count> | <topics> |
| NN | Domain | [exams/NN-file.md](exams/NN-file.md) | <check count> | <topics> |
| NN | Challenge title | [challenges/NN-file.md](challenges/NN-file.md) | <check count> | <topics> |
```

Topics must match the `Topics:` line in the file.

## 7. Update the topic index in CATALOG.md

For each topic covered, add the number to the matching row. Add a new row if no topic fits.

Exam numbers carry an `E` prefix, challenge numbers a `C` prefix, for example `E01` and `C01`. Keep numbers ascending inside each row: labs, then exams, then challenges.

## 8. Update SKILL.md only if a convention changed

Routine files never touch `SKILL.md`. Update it only when something new appears, for example:

- A new content type or folder
- The file template gains or loses a section
- Numbering or naming rules change

If a convention changes, also update this file.

## 9. Verify

- [ ] No duplicate file exists
- [ ] File is in `labs/`, `exams/`, or `challenges/`, name matches `NN-short-kebab-title.md`
- [ ] Header has Course, title, and `Topics:`
- [ ] Every step or task from the paste is present, in order
- [ ] Exam files: each task has requirement, my solution, official solution, technique, difference
- [ ] Challenge files: each task has requirement, my solution, review with a verdict, what to fix
- [ ] Challenge files: score card at the top, pattern check at the end, no invented official answer
- [ ] Solutions collapsed, review links point at real `labs/` and `exams/` files
- [ ] `exams/CHEATSHEET.md` updated
- [ ] Commands are inside fenced blocks
- [ ] No real credentials stored
- [ ] Table row added, with correct step or check count
- [ ] Topic index rows updated
- [ ] Report to the user: file created, catalog rows changed, anything dropped or redacted
