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

---

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

---

## Output Gate

Validate the generated message:

- Subject is ≤ 40 characters
- Subject matches regex
- Verb matches Verb Selection Table
- Body is present when Body Requirement Table says Yes
- Body contains no violations (self-reference, subjective language, file
  listings, code symbols)

If any check fails: regenerate.

---

## Decision Tables

### Subsystem Resolution

| Priority | Match                                      | Subsystem                                             |
| -------- | ------------------------------------------ | ----------------------------------------------------- |
| 1        | All files share exact deepest subdirectory | directory basename, lowercased                        |
| 2        | Files match known tool pattern (any depth) | tool name                                             |
| 3        | Single file at repo root                   | filename without extension, dots stripped, lowercased |
| 4        | `package.json` detected framework          | framework name                                        |
| 5        | File-type patterns match                   | per inference table                                   |
| 6        | No prior match                             | `core`                                                |

Priority 1 examples:

- `src/components/a.ts` + `src/components/b.ts` → `components`
- `src/components/a.ts` + `src/lib/b.ts` → no match

Priority 3 applies only when commit contains exactly one file at repo root.

---

### Tool Config Patterns

| Pattern                             | Subsystem    |
| ----------------------------------- | ------------ |
| `eslint.config.*`, `.eslintrc.*`    | `eslint`     |
| `.prettierrc*`, `prettier.config.*` | `prettier`   |
| `tsconfig*.json`                    | `typescript` |
| `vite.config.*`                     | `vite`       |
| `tailwind.config.*`                 | `tailwind`   |

---

### Framework Detection

Read `package.json`. First match wins:

| Priority | Match                 | Result      |
| -------- | --------------------- | ----------- |
| 1        | `@sveltejs/kit`       | `sveltekit` |
| 2        | `nuxt` / `nuxt3`      | `nuxt`      |
| 3        | `next`                | `nextjs`    |
| 4        | `@remix-run/react`    | `remix`     |
| 5        | `gatsby`              | `gatsby`    |
| 6        | `astro`               | `astro`     |
| 7        | `solid-js`            | `solid`     |
| 8        | `@angular/core`       | `angular`   |
| 9        | `react` / `react-dom` | `react`     |
| 10       | `vue`                 | `vue`       |
| 11       | `svelte`              | `svelte`    |

No `package.json` match: check root for `next.config.*`, `svelte.config.*`,
`astro.config.*`, `nuxt.config.*`, `gatsby-config.*`, `angular.json`.

No match: no framework.

---

### Inference Table

| Priority | File pattern                              | Subsystem    |
| -------- | ----------------------------------------- | ------------ |
| 1        | `*page.*`, `*route.*`, `*+page.*`         | `routes`     |
| 2        | `*layout.*`, `*+layout.*`                 | `layout`     |
| 3        | `*component.*` or `components/` directory | `components` |
| 4        | `*.d.ts` or `types/` directory            | `typescript` |
| 5        | `*.css`, `*.scss`, `*.sass`, `*.less`     | `css`        |
| 6        | `*.html`, `*.htm`                         | `html`       |

---

### Change-type Resolution (multi-file)

When a commit contains files with different change types, a change type
predicate is satisfied if the type applies to at least one staged file:

- `change_type == "added"` — true when any file is added
- `change_type == "renamed"` — true when any file is renamed
- `change_type == "deleted"` — true when any file is deleted
- Otherwise the effective type is `"modified"`

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

---

### Body Requirement

| Priority | Condition                                               | Body required |
| -------- | ------------------------------------------------------- | ------------- |
| 1        | `change_type == "renamed"`                              | No            |
| 2        | `change_type == "deleted" AND conditional_count == 0`   | No            |
| 3        | `conditional_count > 0`                                 | Yes           |
| 4        | `insertions != deletions AND change_type == "modified"` | Yes           |
| 5        | (fallback)                                              | No            |

---

## Examples

| Subject                                    | Body                                                                                                                                                                                                                                    | Verdict                          |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| `git-commit: force deterministic output`   | `Prompt-sensitive output produces inconsistent message format and length across runs, making history scans unreliable.`                                                                                                                 | pass                             |
| `readme: align project heading`            | —                                                                                                                                                                                                                                       | pass                             |
| `skills: enforce deterministic generation` | `Natural-language descriptions produce variable output across runs and agents, breaking the reproducibility guarantee that a deterministic skill file requires.`                                                                        | pass                             |
| `prettier: set project-wide formatting`    | —                                                                                                                                                                                                                                       | pass                             |
| `routes: harden session hydration`         | `Suspense boundaries in the root layout resolve before middleware populates the session cookie, causing a brief logged-out UI before re-rendering. Move session hydration into the server loader for consistent layout on first paint.` | pass                             |
| `routes: update error handling`            | —                                                                                                                                                                                                                                       | fail — generic verb `update`     |
| `components: add button component`         | —                                                                                                                                                                                                                                       | fail — generic verb `add`        |
| `core: improve performance`                | —                                                                                                                                                                                                                                       | fail — subjective verb `improve` |
| `css: fix layout issue`                    | —                                                                                                                                                                                                                                       | fail — banned verb `fix`         |

---

## Workflow

Workflow steps are executed sequentially. No step is skipped.

### Step 1 — Read Diff

Read `git diff --staged --no-color`. Extract file paths, change types
(added/deleted/renamed/modified), insertion/deletion counts, conditional
additions, and dependency changes in package.json.

If no staged changes: output `no changes staged`, stop.

---

### Step 2 — Resolve Subsystem

Apply Subsystem Resolution decision table. First match wins.

---

### Step 3 — Compose Subject

Format: `<subsystem>: <summary>`

Apply Verb Selection Table. Output of the table is the single verb for the
subject.

Summary: `<verb> <rest-of-summary>`

Apply subject corrections after composing.

---

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

---

### Step 5 — Validate and Deliver

Run Output Gate checks. Output commit message only. No surrounding text.

---

## Validation scenarios

Each scenario lists staged changes and expected output.

| ID  | Staged files                                              | Change nature                      | Expected behavior                                                                                |
| --- | --------------------------------------------------------- | ---------------------------------- | ------------------------------------------------------------------------------------------------ |
| T1  | (none)                                                    | —                                  | Output `no changes staged`. Stop.                                                                |
| T2  | `src/components/Button.tsx`                               | Add aria attributes to button      | Subsystem `components`. Subject ≤ 40 chars. Verb not banned. Pass.                               |
| T3  | `README.md` (repo root)                                   | Update project description         | Subsystem `readme` (priority 3). Subject ≤ 40 chars. Pass.                                       |
| T8  | `src/app/page.tsx`                                        | Validate input on form submit      | Subject `app: validate form input.` → remove trailing period → `app: validate form input`. Pass. |
| T9  | `src/components/Header.tsx` + `src/components/Footer.tsx` | Refactor shared nav styles         | Subsystem `components` (priority 1, same directory). Subject ≤ 40 chars. Verb not banned. Pass.  |
| T10 | `src/components/a.ts` + `src/lib/b.ts`                    | Various fixes                      | No priority 1 match (different dirs). Falls to next priority.                                    |
| T11 | `src/core/config.ts`                                      | Rename env variable for clarity    | Trivial change. Subject alone explains. No body. Pass.                                           |
| T12 | `src/api/handler.ts`                                      | Add rate limiting to prevent abuse | Non-obvious change. Body present explaining why (abuse prevention). Pass.                        |
