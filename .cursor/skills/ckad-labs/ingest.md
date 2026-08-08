# Ingest a new lab

Follow every step. Do not skip the duplicate check or the catalog updates.

Run this only when the user explicitly asks to save or create a lab file.

## 1. Duplicate check

Grep the workspace root for the lab title and for one distinctive command from the pasted text.

If a lab file already covers it, tell the user which file and stop. Do not create a second file.

## 2. Pick the number

List `NN-*.md` in the workspace root. Take the next free number. Zero-pad to two digits, for example `13`.

Numbers follow the order the user pastes labs, not the course portal order.

## 3. Name the file

`NN-short-kebab-title.md`

- Three to six words in the title
- Lowercase, hyphens, no spaces
- Drop filler words from the course title

## 4. Write the file

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

Rules for the content:

- Multi-step labs stay in one file. One `### Step N - <name>` per lab step.
- Keep the user's commands, flags, and wording. Fix only structure, formatting, and clear typos.
- Put every command in a fenced block: `bash` for shell, `yaml` for manifests.
- Keep inline notes that explain placeholders, for example `...` replaced by tab completion.
- Move repeated warnings and definitions into `## Notes` so the steps stay short.
- Never store real credentials. Replace a password or key with a pointer to the lab UI.
- Reuse existing tag names in `Topics:` where they fit. Check the topic index in `CATALOG.md`.

## 5. Update the labs table in CATALOG.md

Add a row in number order:

```markdown
| NN | Lab title | [NN-file.md](NN-file.md) | <step count> | <topics> |
```

Topics must match the `Topics:` line in the file.

## 6. Update the topic index in CATALOG.md

For each topic the lab covers, add the number to the matching row. Add a new row if no topic fits.

Keep numbers in ascending order inside each row.

## 7. Update SKILL.md only if a convention changed

Routine labs never touch `SKILL.md`. Update it only when something new appears, for example:

- A lab needs extra assets, so it gets a subfolder
- The file template gains or loses a section
- Numbering or naming rules change

If a convention changes, also update this file.

## 8. Verify

- [ ] No duplicate lab file exists
- [ ] File name matches `NN-short-kebab-title.md`
- [ ] Header has Course, Lab, and `Topics:`
- [ ] Every step from the paste is present, in order
- [ ] Commands are inside fenced blocks
- [ ] No real credentials stored
- [ ] Labs table row added, with correct step count
- [ ] Topic index rows updated
- [ ] Report to the user: file created, catalog rows changed, anything dropped or redacted
