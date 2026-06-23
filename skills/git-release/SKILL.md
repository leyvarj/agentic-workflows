---
name: git-release
description:
  Deterministic release versioning and changelog generator from git log.
  Produces semver bumps and subsystem-grouped changelog from commit history.
compatibility: opencode, claude code
metadata:
  audience: ai agents
  platform: github, gitea, gitlab, forgejo
---

## Role

Generate exactly one release version and changelog from the git log between the
latest tag and HEAD.

Only `git describe --tags --abbrev=0 --first-parent`,
`git tag -l <tag> --format='%(contents)'`,
`git log <tag> --format="%s%n%b" --no-color -1`, and
`git log <tag>..HEAD --first-parent --format="%s%n%b" --no-color` are read.
Uncommitted changes are ignored.

If the latest tag points to HEAD:

```
no new commits since <tag>
```

Stop.

## Grammar

### Output grammar

```
release-message  = version-line "\n\n" summary "\n\n" sections
version-line     = "v" version-num
version-num      = major "." minor "." patch
summary          = paragraph, ≤ 4 lines, wrap ≤ 72 characters
sections         = section | sections "\n" section
section          = subsystem-name ":" "\n\n" entries
subsystem-name   = word
entries          = entry | entries "\n" entry
entry            = "- " verb-description
verb-description = present-tense, no trailing period, ≤ 72 characters
```

### Version validation

| Match                                          | Action     |
| ---------------------------------------------- | ---------- |
| `version-num` does not match `^\d+\.\d+\.\d+$` | regenerate |
| `version-num` equals `0.0.0`                   | regenerate |

## Output Gate

Validate the generated message:

- `version-num` matches `^\d+\.\d+\.\d+$`, not `0.0.0`
- Summary is ≤ 4 lines
- Summary wraps at ≤ 72 characters
- No section has zero entries
- Summary and entries contain no self-reference, subjective language, or exact
  code names
- Each entry description starts with a lowercase letter

If any check fails: regenerate.

## Decision Tables

### Version Resolution

| Priority | Condition                                  | Bump    |
| -------- | ------------------------------------------ | ------- |
| 1        | any commit body contains `BREAKING CHANGE` | `major` |
| 2        | any commit verb is `implement`             | `minor` |
| 3        | (fallback)                                 | `patch` |

If no prior tags exist: initial release, version is `0.1.0`.

### Commit Parsing

Parse each commit subject against the Linux kernel commit style. First match
wins.

| Priority | Pattern               | Subsystem | Description          |
| -------- | --------------------- | --------- | -------------------- |
| 1        | `^(subsystem): (.+)$` | extracted | extracted after `: ` |
| 2        | (fallback)            | `core`    | full commit subject  |

In the pattern, `subsystem` and the description are captured groups —
`subsystem` is the word before `: `, and the description is everything after
`: `. The first word of the description is the verb.

Description uses present tense, no trailing period, ≤ 72 characters. If longer,
truncate from the end.

### Section Display Order

Sections appear in alphabetical order by subsystem name. Omitted entirely if
empty.

## Workflow

Workflow steps are executed sequentially. No step is skipped.

### Step 1 — Read Git Log

Run `git describe --tags --abbrev=0 --first-parent`. If it prints
`fatal: No names found` or the output is empty, no tags exist — treat as initial
release.

If a tag exists:

1. Capture the **previous release context**:
   - Run `git tag -l <tag> --format='%(contents)'` to read the tag annotation.
   - If the output is non-empty, use it as the previous release context.
   - If empty (lightweight tag with no annotation), run
     `git log <tag> --format="%s%n%b" --no-color -1` and use that instead.
2. Read `git log <tag>..HEAD --first-parent --format="%s%n%b" --no-color`. If
   the output is empty (tag points to HEAD), output
   `no new commits since <tag>`, stop.

Initial release (no tags found): read
`git log --first-parent --format="%s%n%b" --no-color` (all commits on the main
branch ancestry). No previous release context is available.

If `git log` output is empty at this point (no commits in the repository):
output `no commits found`, stop.

Extract from each commit: subsystem, description, and verb (first word of
description) using Commit Parsing table. Check commit body for `BREAKING CHANGE`
indicators.

### Step 2 — Resolve Version

If initial release (no prior tags): version is `0.1.0`. The Version Resolution
table does not apply — skip to Step 3.

Otherwise, apply Version Resolution table to determine the bump type.

Compute the new semver version from the latest tag. Strip the leading `v` if
present. Strip any pre-release suffix (from the first `-` onwards). If the
remaining version is not semver-compatible:

```
latest tag <tag> is not a valid semver version
```

Stop.

Increment the appropriate segment and reset all lower segments to zero. Do not
re-attach the pre-release suffix — this release marks the first stable version
since the pre-release phase.

### Step 3 — Compose Summary

Compose a paragraph summarizing the collective impact of this release.
Synthesize individual commits into a cohesive narrative.

If previous release context is available, use it to understand the project's
trajectory. Position this release as a continuation: describe what is now
possible or resolved that the previous release did not cover. This ensures
continuity without duplicating the previous changelog.

If no previous release context (initial release): summarize what the project can
do for the first time.

Summary rules:

- explains _why_ this release matters
- not what individual commits mechanically did
- no self-reference
- no subjective or emotional language
- no file listings, bullet points, changelog formatting, footers
- file types, module names, domain concepts are valid references

### Step 4 — Group and Format Changelog

Group commits by subsystem. Render sections in Display Order (omit empty
sections). Within each section, sort entries by commit order.

Format each entry as: `- <verb-description>`

Changelog rules:

- description uses present tense
- no trailing period
- no self-reference
- no subjective or emotional language
- no exact function, variable, class, or parameter names
- entries wrap at ≤ 72 characters

### Step 5 — Validate and Deliver

Run Output Gate checks. Output release version and changelog only. No
surrounding text.
