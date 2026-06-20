---
name: git-commit
description:
  Deterministic commit message generator with dynamic subsystem resolution.
  Enforces imperative verb-led subjects (what) with explanatory bodies (why).
compatibility: opencode, claude code
metadata:
  audience: ai agents
  platform: github, gitea, gitlab, forgejo
---

## Role

Generate exactly one commit message from staged diffs.

Only `git diff --staged --no-color` is read. Unstaged changes are ignored.

If no staged changes exist:

```
no changes staged
```

Stop.

## Grammar

### Output grammar

```
message       = subject [ "\n\n" body ]
subject       = subsystem ": " summary
subject       = ≤ 40 characters total
summary       = imperative-verb lowercase-text
body          = paragraph, ≤ 8 lines, wrap ≤ 72 characters
```

### Subject validation

Subject must match:

```
^[a-z]+(-[a-z]+)*(/[a-z]+(-[a-z]+)*)?: [a-z].+$
```

Subject must be ≤ 40 characters total.

| Match                                 | Action                     |
| ------------------------------------- | -------------------------- |
| summary ends with period              | remove trailing period     |
| subsystem starts with uppercase       | lowercase entire subsystem |
| missing colon + space after subsystem | insert `: `                |
| summary starts with uppercase         | lowercase first word       |
| subject exceeds 40 characters         | truncate summary from end  |

## Output Gate

Validate the generated message:

- Subject is ≤ 40 characters
- Subject matches regex
- Verb matches Verb Selection Table
- Body is present when Body Requirement Table says Yes
- Body contains no violations (self-reference, subjective language, file
  listings, code symbols)

If any check fails: regenerate.

## Decision Tables

### Subsystem Resolution

| Priority | Condition                                     | Subsystem                                    |
| -------- | --------------------------------------------- | -------------------------------------------- |
| 1        | All files share a deepest common subdirectory | that directory name, lowercased              |
| 2        | Single file at repo root                      | filename stem (before first `.`), lowercased |
| 3        | (fallback)                                    | `core`                                       |

Examples: `README.md` at root → `readme`. `my-component.config.ts` at root →
`my-component`.

### Tool Config Patterns

| Pattern                             | Subsystem    |
| ----------------------------------- | ------------ |
| `eslint.config.*`, `.eslintrc.*`    | `eslint`     |
| `.prettierrc*`, `prettier.config.*` | `prettier`   |
| `tsconfig*.json`                    | `typescript` |
| `vite.config.*`                     | `vite`       |
| `tailwind.config.*`                 | `tailwind`   |
| `vitest.config.*`                   | `vitest`     |
| `jest.config.*`                     | `jest`       |

### Change-type Resolution (multi-file)

change_type is `"added"` if any file is added, `"renamed"` if any file is
renamed, `"deleted"` if any file is deleted, otherwise `"modified"`.

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

### Body Requirement

| Priority | Condition                                               | Body required |
| -------- | ------------------------------------------------------- | ------------- |
| 1        | `change_type == "renamed"`                              | No            |
| 2        | `change_type == "deleted" AND conditional_count == 0`   | No            |
| 3        | `conditional_count > 0`                                 | Yes           |
| 4        | `insertions != deletions AND change_type == "modified"` | Yes           |
| 5        | (fallback)                                              | No            |

## Workflow

Workflow steps are executed sequentially. No step is skipped.

### Step 1 — Read Diff

Read `git diff --staged --no-color`. Extract: paths, change types, +/- counts,
conditional_count, dependency_changes.

`conditional_count` = number of staged lines introducing `if`, `else`, `switch`,
`case`, ternary (`? :`), nullish coalescing (`??`), or short-circuit (`&&`,
`||`) guard patterns.

`dependency_changes` = number of lines in `package.json` that add, remove, or
update a dependency entry.

If no staged changes: output `no changes staged`, stop.

### Step 2 — Resolve Subsystem

Apply Subsystem Resolution decision table. First match wins. If any staged file
matches a Tool Config Pattern, that pattern's subsystem overrides Priority 2
and 3.

Priority 1 example: `src/components/a.ts` + `src/components/b.ts` → `components`

Priority 2 example: `README.md` (repo root, no other files) → `readme`

### Step 3 — Compose Subject

Format: `<subsystem>: <summary>`

Apply Verb Selection Table to select the verb.

Summary: `<verb> <rest-of-summary>`

Apply subject corrections after composing.

### Step 4 — Compose Body

Apply Body Requirement Table. If Yes: compose body. If No: skip.

Body rules:

- explains _why_ (reasoning, constraint, trade-off)
- not what the diff mechanically does
- no self-reference
- no subjective or emotional language
- no file listings, bullet points, changelog formatting, footers
- file types, module names, domain concepts are valid references
- no exact function, variable, class, or parameter names

### Step 5 — Validate and Deliver

Run Output Gate checks. Output commit message only. No surrounding text.
