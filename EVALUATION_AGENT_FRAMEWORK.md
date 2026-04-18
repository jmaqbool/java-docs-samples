# Evaluation Agent Framework

A generic, reusable framework for evaluating the behavior of a primary ("subject")
agent against objectives that are **derived automatically from the project's own
design doc and code** — not from a hand-written rubric that drifts out of date.

The evaluation agent runs continuously alongside normal development: whenever
new work is assigned to the subject agent, the evaluator re-reads the current
specs, re-derives objectives, and grades the subject's output against them.

---

## 1. Goals

1. **Specs are the source of truth.** The design doc and the code define what
   "correct" means. The evaluator extracts criteria from them, so the rubric
   evolves as the project evolves.
2. **Generic.** Nothing in the evaluator is hard-coded to a specific feature
   area. A new project only needs to point the evaluator at its design doc and
   source tree.
3. **Continuous.** The evaluator is invoked automatically on each new task
   assignment (or PR, or merge), not as a one-off review.
4. **Coverage-aware.** The evaluator checks both the **happy path** (does the
   subject follow the documented patterns?) and **edge cases** (does it handle
   the failure modes the design doc and code actually account for?).

## 2. Inputs and Outputs

### Inputs
- **Design doc(s)** — Markdown / text specifications describing intended
  behavior, invariants, and edge cases.
- **Code** — Current source tree. Used both as a spec (existing patterns,
  public APIs, error handling) and as context for the subject's changes.
- **Subject output** — The artifact produced by the primary agent for a given
  task (diff, file, response, trace).
- **Task description** — What the subject was asked to do.

### Outputs
- **Objective list** — Extracted criteria, each tagged with its source
  (doc section, file:line).
- **Per-objective verdict** — pass / fail / not-applicable, with evidence.
- **Edge-case coverage report** — which documented edge cases the subject
  handled, which it ignored.
- **Overall verdict** — ship / revise / block, plus a short rationale.

## 3. Pipeline

```
 ┌──────────────┐   ┌────────────────────┐   ┌───────────────────┐
 │ Task assigned│──▶│ 1. Extract specs    │──▶│ 2. Derive         │
 └──────────────┘   │    (doc + code)    │   │    objectives     │
                    └────────────────────┘   └─────────┬─────────┘
                                                       │
 ┌──────────────┐   ┌────────────────────┐             ▼
 │ Subject's    │──▶│ 3. Validate subject│◀──── objectives + evidence
 │ output       │   │    against each    │
 └──────────────┘   │    objective       │
                    └─────────┬──────────┘
                              ▼
                    ┌────────────────────┐
                    │ 4. Report + gate   │
                    └────────────────────┘
```

### Step 1 — Extract specs
- Parse the design doc into sections; keep section headings as anchors.
- Walk the code to catalog patterns the evaluator should respect: public
  APIs, naming conventions, error types, logging conventions, test layout.
- Surface explicit invariants (`assert`, precondition checks) and documented
  edge cases (comments, doc sections titled "Edge cases" / "Errors" / etc.).

### Step 2 — Derive objectives
Each objective is a structured record:

```jsonc
{
  "id": "OBJ-007",
  "source": "docs/design.md#retries",
  "statement": "Retries use exponential backoff capped at 5 attempts.",
  "kind": "behavior" | "pattern" | "edge-case" | "invariant",
  "checks": [
    { "type": "code-contains", "pattern": "backoff\\s*=.*exponential" },
    { "type": "semantic",      "prompt": "Does the change retry with exponential backoff, max 5?" }
  ]
}
```

Objectives are **re-derived on every run** so that spec edits propagate
immediately — there is no separate rubric to maintain.

### Step 3 — Validate
For each objective, run its checks against the subject's output:
- **Static checks** — regex, AST queries, type checks, test execution.
- **Semantic checks** — delegate to an LLM with the objective statement, the
  relevant spec excerpt, and the subject's diff. The LLM returns
  pass/fail + one-sentence evidence.

### Step 4 — Report and gate
- Emit a machine-readable report (JSON) and a human summary (Markdown).
- Gate on: any failed objective tagged `invariant`, or any un-covered
  edge case the design doc explicitly calls out.
- Non-gating findings are surfaced as review comments.

## 4. Continuous Operation

The evaluator is wired into the task lifecycle:

| Trigger                      | Action                                              |
| ---------------------------- | --------------------------------------------------- |
| New task assigned to subject | Snapshot current design doc + code → cache specs.   |
| Subject produces output      | Run pipeline (Steps 1–4) on the diff.               |
| Design doc changes           | Invalidate cached objectives; re-derive on next run.|
| Code convention change       | Same — cache key includes a hash of spec sources.   |

Typical wiring points:
- Git pre-push / CI job on the subject's PR branch.
- Chat/agent orchestrator hook: run before marking a task "done".

## 5. Directory Layout (convention)

```
eval-agent/
├── config.yaml           # pointers to design docs, code roots, gating rules
├── extractors/
│   ├── doc_extractor.py  # design doc → objectives
│   └── code_extractor.py # code → patterns + invariants
├── checks/
│   ├── static.py         # regex / AST / tests
│   └── semantic.py       # LLM-backed checks
├── runner.py             # orchestrates the 4-step pipeline
└── reports/              # per-run JSON + Markdown
```

## 6. Configuration

`config.yaml` is the only per-project file that should change:

```yaml
design_docs:
  - docs/design.md
  - docs/edge-cases.md

code_roots:
  - src/

gating:
  fail_on:
    - kind: invariant
    - kind: edge-case
      when: documented
  warn_on:
    - kind: pattern

llm:
  model: claude-opus-4-7
  max_tokens: 2048
```

## 7. Extending

- **New objective kinds** — add a discriminator value and a check handler.
- **New extractors** — plug in for non-Markdown specs (ADRs, proto files,
  OpenAPI) by emitting the same objective record shape.
- **New gates** — modify `gating` in `config.yaml`; no code changes required
  for common policies.

## 8. Non-goals

- Replacing human review for design-level decisions.
- Judging code quality beyond what the specs document.
- Enforcing style rules that belong in a linter.

---

*This framework is intentionally minimal: the design doc and the code already
describe what "good" looks like. The evaluation agent's job is to read them
faithfully and keep the subject agent honest against that standard.*
