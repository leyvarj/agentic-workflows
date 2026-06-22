---
name: git-issue
description:
  Template-based issue body from user descriptions. Formats user-provided
  details into a structured bug report.
compatibility: opencode, claude code
metadata:
  audience: ai agents
  platform: github, gitea, gitlab, forgejo
---

## Role

Generate exactly one bug report issue from the user's description only.

No git context is read. Every issue is formatted solely from the user's stated
input.

If no description is provided:

```
no description provided
```

Stop.

## Grammar

### Output grammar

```
issue-message = title "\n\n" body
title         = ≤ 50 characters, no trailing period
title         = noun-phrase describing the bug
body          = description "\n\n"
               steps-block "\n\n"
               expected-line "\n"
               actual-line "\n"
               environment-line

description      = paragraph, ≤ 8 lines, wrap ≤ 72
steps-block      = "Steps to reproduce:\n" bullet-steps
bullet-steps     = bullet-step ( "\n" bullet-step )*
bullet-step      = "- " step-description
expected-line    = "Expected: " sentence
actual-line      = "Actual: " sentence
environment-line = "Environment: " details
```

### Title validation

| Match                       | Action                      |
| --------------------------- | --------------------------- |
| title ends with period      | remove trailing period      |
| title starts with lowercase | uppercase first character   |
| title exceeds 50 characters | regenerate with compression |

## Output Gate

Validate the generated message:

- Title is ≤ 50 characters
- Title has no trailing period
- Title starts with uppercase
- Body follows the output grammar exactly
- Body contains no violations (self-reference, subjective language, file
  listings in description)
- Body wraps at ≤ 72 characters

If any check fails: regenerate.

## Workflow

Workflow steps are executed sequentially. No step is skipped.

### Step 1 — Read Input

Read the user's issue description. Extract description text.

If no description: output `no description provided`, stop.

### Step 2 — Compose Title

Compose a noun-phrase title describing the bug. Apply title corrections after
composing.

### Step 3 — Compose Body

Bug body only:

- compose description paragraph explaining the problem
- compose "Steps to reproduce:" followed by bullet list of steps
- compose "Expected: <expected behavior>"
- compose "Actual: <actual behavior>"
- compose "Environment:" followed by relevant details

Body rules:

- explains _what_ the bug is
- explains expected vs actual behavior
- no self-reference
- no subjective or emotional language
- no footnotes
- no file listings in the description paragraph
- file types, module names, domain concepts are valid references
- code symbols are valid when necessary for explanation
- body wraps at ≤ 72 characters
- body structure follows the output grammar exactly
- description is derived solely from the user's input

### Step 4 — Validate and Deliver

Run Output Gate checks. Output issue title and body only. No surrounding text.
