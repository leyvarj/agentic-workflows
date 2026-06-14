---
name: create-skill
description:
  Generate a deterministic SKILL.md from a natural-language skill description
compatibility: opencode, claude code
metadata:
  audience: ai agents
scores:
  deterministic: 10
  outputStrictness: 10
  semanticAmbiguity: 10
  multiAgentConsistency: 10
  executability: 10
---

# Create Skill

## Role

Generate exactly one `SKILL.md` document from a user-provided skill description.

The generated document must conform to the required structure, grammar,
workflow, validation gates, decision tables, and state machine defined in this
specification.

Delivery is permitted only when every validation gate passes.

---

## 5 Axes

### Deterministic flow

Every decision must be represented by:

- an ordered workflow step
- a first-match decision table
- a named state transition

No decision may be represented solely through prose.

Prohibited phrases:

```text
if appropriate
as needed
best effort
best-effort
if desired
if applicable
if necessary
```

---

### Output constraint strictness

Output must:

1. conform to the required document grammar
2. pass all output gates
3. pass all validation checks

No alternative output structure is permitted.

---

### Semantic ambiguity resistance

All branching must use:

- explicit lookup tables
- explicit state transitions
- explicit validation rules

Prohibited constructs:

```text
detect whether
determine if
assess whether
check if the user intends
infer
interpret
evaluate intent
```

---

### Multi-agent consistency

For identical input:

- identical workflow order must be used
- identical validation order must be used
- identical scoring rules must be used

Subjective judgments are prohibited.

All decisions must be traceable to:

- a table row
- a workflow step
- a validation rule

---

### Executability as a specification

The specification must contain:

- one start state
- one terminal state
- named transitions
- bounded iteration counts

Infinite loops are prohibited.

Unreachable states are prohibited.

---

## Grammar

### Required SKILL.md grammar

```bnf
skill =
    frontmatter
    role
    axes
    grammar
    workflow
    decision_tables
    self_review
    anti_patterns
    state_machine

frontmatter =
    yaml_frontmatter

role =
    "## Role"
    paragraph

axes =
    "## 5 Axes"
    axis_definition+

axis_definition =
    "###"
    axis_name
    paragraph

grammar =
    "## Grammar"
    required_structure
    output_gate

required_structure =
    "### Required output structure"
    fenced_bnf_block

output_gate =
    "### Output gate"
    numbered_checks

workflow =
    "## Workflow"
    ordered_steps

decision_tables =
    "## Decision tables"
    first_match_table+

self_review =
    "## Self-review"
    review_table
    violation_report

anti_patterns =
    "## Anti-pattern registry"
    anti_pattern_table

state_machine =
    "## State machine"
    states
    transitions
```

### First-match table grammar

```bnf
table =
    header
    row+

header =
    "| Priority | Match | Action |"

row =
    "| integer | pattern | action |"
```

### First-match evaluation

Evaluation order:

1. Sort by Priority ascending
2. Evaluate row 1
3. First matching row wins
4. Execute Action
5. Stop evaluation

No subsequent rows may execute.

---

## Output Gate

The generated SKILL.md must pass all checks.

### Gate 1

Required sections:

- Role
- 5 Axes
- Grammar
- Workflow
- Decision tables
- Self-review
- Anti-pattern registry
- State machine

### Gate 2

Frontmatter contains:

```yaml
scores:
  deterministic: 10
  outputStrictness: 10
  semanticAmbiguity: 10
  multiAgentConsistency: 10
  executability: 10
```

### Gate 3

At least one first-match decision table exists.

### Gate 4

At least one state machine exists.

### Gate 5

Exactly one terminal state exists.

### Gate 6

All loops have explicit iteration limits.

### Gate 7

Prohibited regexes must not appear outside:

- code fences
- self-review tables
- anti-pattern registry

Regex list:

```regex
if appropriate
as needed
best effort
best-effort
if desired
if applicable
if necessary
detect whether
determine if
assess whether
check if the user
infer
interpret
evaluate intent
```

### Gate 8

No validation failures remain.

If any gate fails:

- output is forbidden
- regeneration is required

---

## Workflow

### Step 1 — Receive Input

Receive the user skill description.

Output:

```text
INPUT_RECEIVED
```

---

### Step 2 — Scope Extraction

Generate:

- skill name
- skill purpose
- execution target

Output:

```text
SCOPE_DEFINED
```

---

### Step 3 — Grammar Construction

Generate:

- document grammar
- output gates

Output:

```text
GRAMMAR_DEFINED
```

---

### Step 4 — Decision Construction

Convert every decision point into:

- first-match tables

Output:

```text
DECISIONS_DEFINED
```

---

### Step 5 — State Machine Construction

Generate:

- states
- transitions
- terminal state

Output:

```text
STATE_MACHINE_DEFINED
```

---

### Step 6 — Validation

Run validation checks.

Output:

```text
VALIDATED
```

---

### Step 7 — Self Review

Run violation scan.

Output:

```text
REVIEWED
```

---

### Step 8 — Delivery

Deliver SKILL.md.

Output:

```text
DELIVERED
```

---

## Decision Tables

### Document Type Mapping

| Priority | Match                     | Action             |
| -------- | ------------------------- | ------------------ |
| 1        | skill description present | generate SKILL.md  |
| 2        | skill description absent  | validation failure |

---

### Validation Routing

| Priority | Match                  | Action   |
| -------- | ---------------------- | -------- |
| 1        | zero violations        | continue |
| 2        | one or more violations | iterate  |

---

### Iteration Routing

| Priority | Match                | Action     |
| -------- | -------------------- | ---------- |
| 1        | iteration_count < 3  | regenerate |
| 2        | iteration_count >= 3 | fail       |

---

### Section Validation Table

| Priority | Match                  | Action              |
| -------- | ---------------------- | ------------------- |
| 1        | role_section_absent    | add_role_section    |
| 2        | axes_sections_absent   | add_axes_sections   |
| 3        | grammar_section_absent | add_grammar_section |
| 4        | workflow_steps_absent  | add_workflow_steps  |
| 5        | decision_tables_absent | add_decision_tables |
| 6        | self_review_absent     | add_self_review     |
| 7        | anti_patterns_absent   | add_anti_patterns   |
| 8        | state_machine_absent   | add_state_machine   |
| 9        | all_sections_present   | proceed_to_gate_3   |

---

### Input Validation Table

| Priority | Match              | Action                |
| -------- | ------------------ | --------------------- |
| 1        | input_is_empty     | request_new_input     |
| 2        | input_is_null      | validation_failure    |
| 3        | input_is_malformed | request_clarification |
| 4        | input_is_valid     | proceed_to_scope      |

---

## Self-review

### Violation Report

Generate:

| ID  | Axis | Location | Violation | Fix |
| --- | ---- | -------- | --------- | --- |

Rules:

1. Scan entire document
2. Record every violation
3. Count violations by axis
4. Calculate score

### Scoring Formula

```text
axis_score = 10 - violation_count
minimum = 0
maximum = 10
```

### Delivery Rule

```text
all axis scores = 10
```

Required.

Any score below 10:

```text
REGENERATE
```

---

## Anti-pattern Registry

| Anti-pattern                      | Failure Mode         | Replacement             |
| --------------------------------- | -------------------- | ----------------------- |
| Natural-language intent detection | Model variance       | Lookup table            |
| Description-based matching        | Ambiguous routing    | Pattern matching        |
| Implicit state transitions        | Non-executable flow  | Explicit transitions    |
| Unbounded loops                   | Infinite execution   | Max iteration count     |
| Subjective scoring                | Inconsistent output  | Violation counting      |
| Freeform branching                | Non-determinism      | First-match tables      |
| Missing terminal state            | Undefined completion | Explicit terminal state |

---

## State Machine

### States

```text
START
  |
  v
INPUT_RECEIVED
  |
  v
SCOPE_DEFINED
  |
  v
GRAMMAR_DEFINED
  |
  v
DECISIONS_DEFINED
  |
  v
STATE_MACHINE_DEFINED
  |
  v
VALIDATED
  |
  v
REVIEWED
  |
  +------------------------+
  | violations > 0         |
  v                        |
ITERATE -------------------+
  |
  | iteration_count < 3
  v
DECISIONS_DEFINED

REVIEWED
  |
  | violations = 0
  v
DELIVERED
  |
  v
END

ITERATE
  |
  | iteration_count >= 3
  v
FAILED
  |
  v
END
```

### Transitions

| From                  | Condition                              | To                    |
| --------------------- | -------------------------------------- | --------------------- |
| START                 | input received                         | INPUT_RECEIVED        |
| INPUT_RECEIVED        | scope extraction finished              | SCOPE_DEFINED         |
| SCOPE_DEFINED         | bnf_structure_validated                | GRAMMAR_DEFINED       |
| GRAMMAR_DEFINED       | all_decisions_mapped                   | DECISIONS_DEFINED     |
| DECISIONS_DEFINED     | state machine complete                 | STATE_MACHINE_DEFINED |
| STATE_MACHINE_DEFINED | validation complete                    | VALIDATED             |
| VALIDATED             | review start                           | REVIEWED              |
| REVIEWED              | violations = 0                         | DELIVERED             |
| REVIEWED              | violations > 0 and iteration_count < 3 | ITERATE               |
| ITERATE               | regenerate                             | DECISIONS_DEFINED     |
| ITERATE               | iteration_count >= 3                   | FAILED                |
| DELIVERED             | complete                               | END                   |
| FAILED                | complete                               | END                   |

### Terminal State

```text
END
```

Exactly one terminal state is permitted.
