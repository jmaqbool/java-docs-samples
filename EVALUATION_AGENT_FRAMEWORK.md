# Evaluation Agent Framework

A generic, reusable evaluation agent that is **called by the working agent(s)**
during a task. It extracts evaluation objectives directly from the project's
design doc and code, evaluates the working agent's current output against
them, and returns **structured, actionable feedback** so the caller can
correct what needs correcting and try again.

The evaluation agent does not commit, gate, or block. It is a tool the
working agent invokes — the working agent remains in charge of acting on
the feedback.

---

## 1. Role in the Workflow

```
 ┌────────────────┐                                 ┌────────────────────┐
 │  Working agent │ ── task + current output ──▶   │ Evaluation agent   │
 │  (caller)      │                                 │  (this framework)  │
 │                │ ◀── structured feedback ──     │                    │
 └────────┬───────┘                                 └────────────────────┘
          │
          ▼
   apply corrections,
   optionally re-call
   until feedback is clean
```

- The **working agent** decides when to call (after a draft, after a
  subtask, before declaring done).
- The **evaluation agent** reads the current design doc + code, derives
  objectives on the fly, checks the caller's output, and returns feedback.
- The **working agent** consumes the feedback, fixes issues, and may call
  again. Iteration continues until the caller is satisfied.

## 2. Inputs and Outputs

### Inputs (from the calling agent)
- `task` — what the caller is trying to accomplish.
- `output` — the caller's current artifact (diff, file, response, plan).
- `context` *(optional)* — extra files or excerpts the caller wants checked.
- `focus` *(optional)* — narrow the evaluation ("only check edge cases",
  "only check retry semantics").

### Specs (read by the evaluator itself)
- **Design doc(s)** — the authoritative statement of intent.
- **Code** — existing patterns, public APIs, invariants, error handling.

### Output (returned to the caller)
A single structured response the caller can parse and act on:

```jsonc
{
  "summary": "2 issues found, 1 suggestion.",
  "findings": [
    {
      "id": "F-1",
      "severity": "must-fix" | "should-fix" | "suggestion",
      "objective": "Retries use exponential backoff capped at 5 attempts.",
      "source": "docs/design.md#retries",
      "evidence": "doRetry() retries forever; no cap in src/Client.java:88.",
      "suggested_action": "Add maxAttempts=5 and exponential delay; see pattern in src/Http.java:42."
    }
  ],
  "covered_edge_cases": ["timeout", "429 rate-limit"],
  "missing_edge_cases":  ["connection reset mid-stream"],
  "ready": false
}
```

`ready: true` means the evaluator found nothing the caller must address.
The caller is still the one who decides what to do with that.

## 3. Pipeline (per call)

1. **Extract specs (fresh each call).**
   Parse the current design doc(s) and scan the relevant code for
   patterns, invariants, and documented edge cases. No cached rubric —
   specs are always the latest truth.
2. **Derive objectives.**
   Produce a list of objectives tagged by kind
   (`behavior`, `pattern`, `edge-case`, `invariant`) and by source
   (`file#section` or `file:line`).
3. **Evaluate the caller's output.**
   For each objective, run the appropriate checks (regex / AST / test
   execution / LLM-backed semantic check) against the caller's output,
   using the caller's `task` and `focus` to prioritize.
4. **Return feedback.**
   Emit the JSON above. Every `must-fix` / `should-fix` finding
   includes an `evidence` field (what went wrong) and a
   `suggested_action` field (what the caller can do about it).

## 4. Calling Contract

The evaluation agent is invoked like any other tool. A minimal call:

```python
feedback = eval_agent.evaluate(
    task="Implement retry policy for HttpClient",
    output=current_diff,
)

for f in feedback["findings"]:
    if f["severity"] == "must-fix":
        apply_fix(f["suggested_action"])

if not feedback["ready"]:
    feedback = eval_agent.evaluate(task=..., output=new_diff)
```

Guidelines for the working agent:
- Call early — a draft is enough; you don't need a final artifact.
- Call again after each correction pass. The evaluator is cheap to re-run.
- Treat `must-fix` as blocking for your own "done" definition.
- Treat `suggestion` as optional; skip with a brief reason if it conflicts
  with the task.

## 5. Behavior Principles

- **Specs are the source of truth.** Objectives come from the design doc
  and the code at the moment of the call — no maintained rubric.
- **Feedback is actionable.** Every finding names what's wrong, where
  the standard comes from, and a concrete next step.
- **No side effects.** The evaluator does not commit, push, open PRs,
  or change files. It only reads and returns feedback.
- **Caller stays in control.** The evaluator advises; the working
  agent decides what to accept, defer, or override.
- **Continuous by invocation.** The evaluator runs whenever the caller
  asks — typically multiple times per task, as work progresses.

## 6. Extension Points

- **New objective kinds** — add a discriminator and a check handler.
- **New spec sources** — plug in extractors for ADRs, proto files,
  OpenAPI, etc., as long as they emit the same objective shape.
- **Focus modes** — pre-built `focus` values (e.g. `edge-cases-only`,
  `api-contract-only`) for callers that want narrow passes.

## 7. Non-goals

- Gating merges, commits, or CI.
- Replacing human review of design-level decisions.
- Writing or modifying code on behalf of the caller.
- Enforcing style rules that belong in a linter.

---

*The evaluation agent is a feedback tool for the agents doing the work.
It reads the specs, evaluates the current draft, and hands back a
structured report so the working agent can correct course — and call
again when it's ready for another look.*
