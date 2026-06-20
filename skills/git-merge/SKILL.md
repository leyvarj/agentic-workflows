---
name: git-merge
description:
  Deterministic merge commit message generator from merged branches. Produces
  fixed-format subject with distilled why-driven body.
compatibility: opencode, claude code
metadata:
  audience: ai agents
  platform: github, gitea, gitlab, forgejo
---

## Role

Generate exactly one merge commit message from `git diff main..HEAD` and the
current branch name.

Only `git diff main..HEAD --no-color` is read. Uncommitted changes are ignored.

If `HEAD` points to `main`:

```
no branch detected
```

Stop.

## Grammar

### Output grammar

```
merge-message  = subject "\n\n" body
subject        = "Merge branch '" branch-name "'"
subject        = fixed format, no substitutions
body           = paragraph, ≤ 4 lines, wrap ≤ 72 characters
```

### Subject validation

| Match                                        | Action     |
| -------------------------------------------- | ---------- |
| subject does not match `^Merge branch '.*'$` | regenerate |
| subject ends with period                     | remove     |

## Output Gate

Validate the generated message:

- Subject matches `^Merge branch '.*'$`
- Body is present (always required)
- Body is ≤ 4 lines
- Body contains no violations (self-reference, subjective language, file
  listings, code symbols)

If any check fails: regenerate.

## Workflow

Workflow steps are executed sequentially. No step is skipped.

### Step 1 — Read Diff and Branch

Read `git diff main..HEAD --no-color`. Read `git diff main..HEAD --stat`. Read
branch name via `git rev-parse --abbrev-ref HEAD`. Extract: branch name, changed
file list, +/- counts.

If `HEAD` is `main`: output `no branch detected`, stop.

### Step 2 — Compose Subject

Format: `Merge branch '<branch-name>'`

No verb selection. No summary. Subject is a fixed template.

### Step 3 — Compose Body

Body is always required.

Compose a distilled summary of why the branch exists. Capture the branch-level
reasoning, constraint, or trade-off. The PR body contains the detailed
description; this body is the condensed version for permanent history.

Body rules:

- explains _why_ (reasoning, constraint, trade-off)
- not what the diff mechanically does
- no self-reference
- no subjective or emotional language
- no file listings, bullet points, changelog formatting, footers
- file types, module names, domain concepts are valid references
- no exact function, variable, class, or parameter names
- body wraps at ≤ 72 characters
- body is ≤ 4 lines

### Step 4 — Validate and Deliver

Run Output Gate checks. Output merge commit message only. No surrounding text.
