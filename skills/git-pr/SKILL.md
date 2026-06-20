---
name: git-pr
description:
  Deterministic PR description generator from branch diffs. Generates
  noun-phrase or imperative titles with why-driven explanatory bodies.
compatibility: opencode, claude code
metadata:
  audience: ai agents
  platform: github, gitea, gitlab, forgejo
---

## Role

Generate exactly one PR title and body from `git diff main..HEAD`.

Only `git diff main..HEAD --no-color` is read. Uncommitted changes are ignored.

If `HEAD` points to `main`:

```
no branch detected
```

Stop.

## Grammar

### Output grammar

```
pr-message        = title [ "\n\n" body ]
title             = ≤ 50 characters, no trailing period
title             = noun-phrase | imperative-verb-led-sentence
noun-phrase       = describes what the PR adds or introduces
imperative        = verb-led sentence describing the change
body              = single-file-body | multi-file-body

single-file-body  = paragraph, ≤ 8 lines, wrap ≤ 72
multi-file-body   = intro-line "\n\n" file-lines "\n\n" summary-paragraph
intro-line        = sentence describing what the PR adds, ends with ":"
file-lines        = one or more file-description lines
file-description  = filename " " purpose-description (wrap ≤ 72)
summary-paragraph = paragraph, ≤ 8 lines, wrap ≤ 72
```

### Title validation

| Match                       | Action                      |
| --------------------------- | --------------------------- |
| title ends with period      | remove trailing period      |
| title starts with lowercase | uppercase first character   |
| title exceeds 72 characters | regenerate with compression |

## Output Gate

Validate the generated message:

- Title is ≤ 50 characters
- Title has no trailing period
- Title starts with uppercase
- When imperative style: verb matches Verb Selection Table
- Body type matches Body Requirement Table
- Body contains no violations (self-reference, subjective language)

If any check fails: regenerate.

## Decision Tables

### Change-type Resolution (multi-file)

change_type is `"added"` if any file is added, `"renamed"` if any file is
renamed, `"deleted"` if any file is deleted, otherwise `"modified"`.

### Title Style Selection

| Priority | Condition                               | Title style   |
| -------- | --------------------------------------- | ------------- |
| 1        | All files have `change_type == "added"` | `noun-phrase` |
| 2        | (fallback)                              | `imperative`  |

### Verb Selection

| Priority | Condition                                           | Verb        |
| -------- | --------------------------------------------------- | ----------- |
| 1        | `change_type == "added" AND conditional_count > 0`  | `harden`    |
| 2        | `change_type == "added" AND dependency_changes > 0` | `bump`      |
| 3        | `change_type == "added"`                            | `implement` |
| 4        | `change_type == "renamed"`                          | `rename`    |
| 5        | `change_type == "deleted"`                          | `remove`    |
| 6        | `conditional_count > 0`                             | `harden`    |
| 7        | `dependency_changes > 0`                            | `bump`      |
| 8        | `deletions > insertions`                            | `remove`    |
| 9        | (fallback)                                          | `correct`   |

Verb Selection applies only when Title Style is `imperative`.

### Body Requirement

| Priority | Condition                 | Body type         |
| -------- | ------------------------- | ----------------- |
| 1        | `file_count > 1`          | `multi-file body` |
| 2        | `conditional_count > 0`   | `single-file`     |
| 3        | `insertions != deletions` | `single-file`     |
| 4        | (fallback)                | None              |

### Body type mapping

| Body type         | Format                                                                       |
| ----------------- | ---------------------------------------------------------------------------- |
| `multi-file body` | Intro line + blank line + file descriptions + blank line + summary paragraph |
| `single-file`     | Single summary paragraph                                                     |
| None              | Omitted                                                                      |

## Workflow

Workflow steps are executed sequentially. No step is skipped.

### Step 1 — Read Diff

Read `git diff main..HEAD --no-color`. Read `git diff main..HEAD --stat`.
Extract: paths, change types, +/- counts, file_count, conditional_count,
dependency_changes.

`conditional_count` = number of staged lines introducing `if`, `else`, `switch`,
`case`, ternary (`? :`), nullish coalescing (`??`), or short-circuit (`&&`,
`||`) guard patterns.

`dependency_changes` = number of lines in `package.json` that add, remove, or
update a dependency entry.

If `HEAD` is `main`: output `no branch detected`, stop.

### Step 2 — Resolve Changes

Apply Change-type Resolution. Compute `file_count`.

### Step 3 — Compose Title

Apply Title Style Selection table.

If `noun-phrase`: compose a noun phrase describing what the PR adds or
introduces.

If `imperative`: apply Verb Selection Table. Format: `<verb> <rest-of-title>`.

Apply title corrections after composing.

### Step 4 — Compose Body

Apply Body Requirement Table.

If `multi-file body`:

- compose intro line ending with `:`
- compose one file-description line per changed file (`filename purpose`)
- compose summary paragraph explaining _why_

If `single-file`: compose summary paragraph explaining _why_.

If None: skip.

Body rules:

- explains _why_ (reasoning, constraint, trade-off)
- not what the diff mechanically does
- no self-reference
- no subjective or emotional language
- file descriptions followed by purpose are valid in multi-file bodies
- code symbols are valid when necessary for explanation
- no markdown formatting, bullet points, or footnotes
- no exact function, variable, class, or parameter names in summary paragraph
- summary paragraph wraps at ≤ 72 characters

### Step 5 — Validate and Deliver

Run Output Gate checks. Output PR title and body only. No surrounding text.
