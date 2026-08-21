---
name: review-ai-agent
description: Repo-type checklist for AI agent and LLM application PRs — prompt changes, tool/function definitions, agent loops, RAG, evals, cost and latency, and prompt-injection safety. Layer on top of review-core.
---

# AI agent / LLM application review checklist

Layered on `review-core`. Severity definitions and the output template come from there.

The distinguishing risk in this repo type: **the failure modes are not in the code path, they are in the model's behaviour.** A diff can be syntactically perfect and still change what the agent does in production. Review the prompt and tool surface with the same seriousness as the code.

## Model agnosticism

Assume the provider will change — for pricing, availability, capability, or a customer requirement. The review question is: **if we had to switch provider next sprint, how many files would this PR force us to touch?** If the answer is more than the adapter layer, say the number.

- Provider SDK called directly from feature code, rather than through the repo's client/adapter interface → MAINTAINABILITY SHOULD-FIX. **BLOCKER** if this PR wires in a *second* provider by a second, parallel route — that's the point where the abstraction stops being recoverable cheaply.
- Model ID hardcoded in code rather than config, env, or DB → SHOULD-FIX (also §3.1 of `review-core`). Model IDs change often enough that a deploy should never be required to swap one.
- Provider-specific response shape parsed outside the adapter (`response.content[0].text`, `choices[0].message`, provider-specific stop reasons or finish states) → SHOULD-FIX; name the adapter method it should go through.
- Provider-specific feature used with no documented fallback — extended thinking, tool-choice semantics, system-prompt handling, prompt caching, structured-output modes, token counting → SHOULD-FIX. State plainly what degrades on another provider rather than blocking the feature.
- Cost or token accounting hardcoded to one provider's pricing table → SHOULD-FIX; pricing belongs in config next to the model registry.
- Retry, rate-limit, and error handling written against one provider's error codes/exception types → SHOULD-FIX; normalise in the adapter.
- Prompt written to exploit one model's known quirk → NIT, but say so explicitly so it is findable when the model changes.

**Not a finding:** a deliberate, documented single-provider decision that the repo already states. Agnosticism is about keeping the seam clean, not about shipping unused abstraction — a second provider that will never exist is over-engineering (see `review-core` §3.2 on premature utils).

## Prompt storage — database by default

**The default is that prompts live in the database**, versioned and changeable without a deploy. A prompt in source code is an exception that must justify itself in writing.

- New prompt added in code **in a repo that already stores prompts in the DB** → **BLOCKER (MAINTAINABILITY)**. Mixed storage means nobody knows where the live text is.
- Prompt in code **without the exemption comment below** → SHOULD-FIX in a repo that has not migrated yet; **BLOCKER** in a repo that has.
- Exemption present but the `reason` is not one of the accepted values, or `detail` is vague ("for now", "simpler", "TODO") → SHOULD-FIX. A reason that could be written about any prompt is not a reason.
- DB-stored prompt with no version history, no rollback, and no record of who changed it → SHOULD-FIX. Editable-without-deploy is only safe if it is also revertable-without-deploy.
- Prompt loaded from the DB on every call with no cache → PERF SHOULD-FIX. No fallback when the DB is unavailable → **BLOCKER** on a production path; a prompt store outage should not be a total outage.
- Prompt changed in the DB while eval fixtures still pin the old text → TESTS SHOULD-FIX; the evals are now measuring something that isn't running.
- Prompt content editable by a role that shouldn't have it (prompts are executable behaviour, not content) → SECURITY SHOULD-FIX.

### Required exemption format

Any prompt that stays in code carries this comment block immediately above it. Keep it exactly this shape so it is greppable across every repo (`grep -rn "PROMPT-IN-CODE"` should produce a complete inventory).

```python
# PROMPT-IN-CODE
# reason: bootstrap
# detail: Runs before the config service is reachable, so no DB read is possible here.
# owner: @chien-hui
# added: 2026-08-21
# revisit: 2027-02-21
SYSTEM_BOOTSTRAP = """..."""
```

```typescript
/**
 * PROMPT-IN-CODE
 * reason: schema-coupled
 * detail: Output keys are generated from the ExportRow type; editing either alone breaks parsing.
 * owner: @data-platform
 * added: 2026-08-21
 * revisit: 2027-02-21
 */
const EXPORT_PROMPT = `...`;
```

**`reason` is a closed vocabulary** — anything outside it is a finding:

| reason | Means | Expected to be permanent? |
|---|---|---|
| `bootstrap` | Needed before the DB or config layer is available. | Yes |
| `safety-critical` | Guardrails or refusal policy that must not change without code review and deploy. | Yes |
| `schema-coupled` | Text is bound to a code-defined schema; changing one alone breaks parsing. | Yes |
| `eval-fixture` | Frozen baseline text used by tests, not by production. | Yes |
| `latency-critical` | Hot path where a prompt-store read misses the budget. `detail` must quote the measured budget. | No — revisit |
| `migration-pending` | Predates the prompt store; scheduled to move. `revisit` must be a real date. | No — revisit |

Reviewer checks on the block: `reason` is from the table; `detail` is specific to *this* prompt; `owner` is a real team or handle; `revisit` is within 6 months for the non-permanent reasons. A `migration-pending` block whose `revisit` date has already passed → SHOULD-FIX, and say how long it has been overdue.

## Prompt changes

- **Any prompt edit with no eval run or no eval result in the PR** → TESTS BLOCKER. A prompt change is a behaviour change shipped without a test. If the repo has no eval harness at all, that is a single PROCESS SHOULD-FIX (raise it once, not per-PR).
- Instruction added that contradicts an existing instruction in the same prompt → CORRECTNESS SHOULD-FIX; quote both lines.
- A specific failure patched with a narrow "do not do X" rule where the general cause is unaddressed → MAINTAINABILITY SHOULD-FIX; prompts accumulate these and rot.
- Few-shot examples that disagree with the stated instructions, or that leak into output format → SHOULD-FIX.
- Output format instructions loosened (e.g. "respond in JSON" softened) while a downstream parser still assumes strictness → **BLOCKER (CORRECTNESS)**.
- Prompt text hardcoded inline where the repo keeps prompts in versioned files → MAINTAINABILITY SHOULD-FIX.
- Real customer data or PII pasted into a prompt, example, or fixture → **BLOCKER (SECURITY)**.

## Tool / function definitions

- **Tool description changed** — this is a behaviour change, treat it like a prompt change: needs an eval. Vague descriptions cause wrong-tool selection.
- Tool schema changed (param added, renamed, made required, enum values changed) without updating the handler, or without checking cached/stored calls → CORRECTNESS BLOCKER.
- A destructive or irreversible tool (delete, send, pay, deploy, write to prod) exposed with no confirmation step, no dry-run, and no scoping → **BLOCKER (SECURITY)**. Say what the worst single tool call could do.
- Tool that accepts a free-form string the model composes and passes to a shell, SQL query, file path, or URL → **BLOCKER (SECURITY)**. The model is an untrusted input source.
- Tool with no timeout, no output size cap, or that can return unbounded text into context → PERF SHOULD-FIX (BLOCKER if it can blow the context window mid-run).
- Error returned to the model with no actionable detail (bare "failed") → the agent will retry blindly. SHOULD-FIX.

## Prompt injection and trust boundaries

- Untrusted content (web page, user upload, email, retrieved document, another agent's output) entering the context **without delimiting and without an instruction that it is data, not instructions** → **BLOCKER (SECURITY)**.
- The lethal-trifecta check: does this PR give one agent run *all three* of (a) access to private data, (b) exposure to untrusted content, (c) the ability to send data outward (HTTP, email, write to a shared surface)? If yes → **BLOCKER (SECURITY)**, and name the exfiltration path.
- Model output used directly as code, a shell command, a redirect target, or rendered as HTML without sanitisation → **BLOCKER (SECURITY)**.
- Agent output rendered as markdown that can carry images/links (data exfiltration via URL) → SECURITY SHOULD-FIX.

## Agent loop and control flow

- Loop with no maximum iteration count, no wall-clock budget, and no token budget → **BLOCKER (PERF)** — this is the runaway-cost bug.
- No termination condition other than the model deciding to stop → BLOCKER.
- Recursive sub-agent spawning with no depth cap → BLOCKER.
- Retry on model error without backoff, or retry of a non-idempotent tool call → SHOULD-FIX / BLOCKER respectively.
- Conversation history appended without truncation or summarisation strategy → PERF SHOULD-FIX; BLOCKER if unbounded on a long-running session.
- Non-determinism assumed away: code that parses model output with a regex or `JSON.parse` and no fallback for a malformed response → CORRECTNESS SHOULD-FIX; BLOCKER on a user-facing path.

## Model, cost and latency

- Model ID changed → state the expected cost and latency delta and whether evals were re-run. Silent model swap → TESTS BLOCKER.
- `max_tokens`, temperature, or thinking budget changed with no stated reason → SHOULD-FIX.
- A new call added inside a loop or per-item, multiplying spend → PERF SHOULD-FIX; BLOCKER if per-row on unbounded input; give a rough per-run cost estimate.
- Prompt cache broken by moving variable content (timestamp, user data) above stable content in the prompt → PERF SHOULD-FIX.
- Streaming removed from a user-facing path → SHOULD-FIX.
- No fallback when the provider errors or rate-limits → SHOULD-FIX; BLOCKER on a production path.

## RAG and retrieval

- Chunking or embedding model changed without a plan to re-index existing vectors → **BLOCKER (DATA)** — the index becomes silently mixed.
- Retrieval with no relevance threshold, so low-quality chunks always fill context → SHOULD-FIX.
- Retrieved documents not scoped to the requesting user/tenant → **BLOCKER (SECURITY)**.
- Citations claimed in output but not verifiably tied to retrieved chunks → SHOULD-FIX.

## Evals and observability

- New capability with no eval case, or a fixed bug with no regression case added → TESTS SHOULD-FIX.
- Eval that only checks the run did not crash, never the quality of the output → NIT, but say what it should assert.
- Agent run not traced (no per-step logging of prompt, tool calls, tokens) where the repo otherwise traces → OBSERVABILITY SHOULD-FIX.
- No token/cost metric emitted for a new call path → SHOULD-FIX.

## Coverage note for this profile

In the **Coverage** section of the output, always state explicitly: *"Prompt behaviour was reviewed by reading, not by execution — eval results are the only real verification."* This keeps the limits of the review honest.
