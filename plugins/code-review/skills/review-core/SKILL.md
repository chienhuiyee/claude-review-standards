---
name: review-core
description: The shared code review contract — severity definitions, deterministic verdict rules, universal engineering rules, and the fixed output template used for every PR review in every repo. Load this before any PR review, then layer the matching repo-type skill (review-web, review-service, review-gateway, review-ai-agent) on top.
---

# Review Core — the shared contract

This skill exists for one reason: **two reviews of two different repos must look identical in structure and apply the same bar.** Repo-type skills add checks. They never change severity definitions, verdict rules, or the output template below.

## 1. Scope of a review

Review **only the diff** plus whatever surrounding code is needed to judge the diff. Do not review pre-existing code the PR did not touch, unless the diff makes an existing bug reachable or worse — in that case say so explicitly and mark it as pre-existing.

Before reviewing, establish:

- The diff: `gh pr diff <n>` (use `--patch`), and the file list with `gh pr view <n> --json files`.
- The PR's stated intent: title + body. If intent is unclear, that is itself a finding (see `PROCESS` category).
- Whether tests changed alongside behaviour.

If the diff is larger than ~1500 changed lines, review it in file groups and say in the output that it was chunked.

## 2. Severity definitions — memorise these, they are the whole point

Every finding gets exactly one severity. Assign by **consequence if merged as-is**, not by how annoying it is.

**BLOCKER** — merging this causes real harm that is hard to reverse. Exactly these qualify:
- Security: injection, authz/authn bypass, secret committed or logged, unsafe deserialisation, SSRF, missing tenant scoping.
- Data loss or corruption: destructive or irreversible migration, unbounded delete/update, lost writes from a race.
- Correctness with user-visible impact on the happy path or a common error path.
- Breaking change to a public contract (API response shape, exported function signature, DB schema, event payload) with no versioning or migration path.
- Production blast radius: unbounded resource use, removal of a rate limit/timeout/circuit breaker, an N+1 or O(n²) on a hot path with realistic data volume.
- A test that was deleted or disabled to make CI pass.
- Anything a repo-type skill or a repo's `overrides.blockers` explicitly names as a blocker.

**SHOULD-FIX** — real defect, but bounded: edge-case bug, missing error handling on an unlikely path, missing test for new branching logic, meaningful perf regression off the hot path, logging/observability gap that would make an incident hard to debug, unclear naming or structure that will cause a future bug.

**NIT** — style, wording, minor duplication, preference. Never affects the verdict.

**Calibration rules (apply strictly, they stop drift):**
- If you are torn between BLOCKER and SHOULD-FIX, ask: *would I roll this back at 2am?* Yes → BLOCKER. No → SHOULD-FIX.
- Do not inflate. A review of a clean PR should return zero blockers and say so plainly.
- Do not pad. If there are no nits, write "None."
- Never report a finding you have not located at a specific `file:line` in the diff.
- Cap NITs at 5. If there are more, note the count and report the top 5.

## 3. Universal engineering rules — every profile, every repo

Repo-type skills add to these; none of them replace these.

### 3.1 No hardcoded literals

User-facing text, error messages and codes, status/type strings, config keys, URLs and endpoints, feature names, and business-rule numbers belong in constants, enums, config, or the i18n catalogue — not inline in a handler or component.

- Literal appearing **2+ times** in the diff or already present elsewhere in the repo → SHOULD-FIX.
- Literal that encodes a **business rule** (threshold, limit, fee, retry count, timeout, page size) → SHOULD-FIX. **BLOCKER** if the same value is duplicated somewhere else, because the two will drift and produce inconsistent behaviour.
- Status / type / role strings compared with `==` where an enum or const exists → SHOULD-FIX. Introducing a new one as a bare string → SHOULD-FIX.
- Any secret, key, host, or credential inline → **BLOCKER (SECURITY)**, already covered in §2.

When flagging, **name the existing constants/enum module the value should go into**. If none exists, say where it should be created. A finding that only says "don't hardcode this" is incomplete.

**Do not be pedantic.** `0`, `1`, `""`, array indices, a one-off log string, test fixtures, and self-explanatory local values are not findings. The test is: *would someone need to change this in two places, or guess what it means?*

### 3.2 Reusability — the utils check (mandatory, bounded procedure)

For every **new standalone function or helper** in the diff that is not intrinsically tied to the component/handler it sits in, run this procedure. Do not skip it and do not speculate.

1. **Locate shared code.** Glob for the repo's shared locations: `**/utils/**`, `**/lib/**`, `**/helpers/**`, `**/common/**`, `**/shared/**`, `internal/pkg/**`, plus whatever the repo actually uses (check imports elsewhere in the diff for the house convention).
2. **Grep for behaviour, not name.** Search 2–3 plausible spellings of the operation *and* a distinctive token from the body (a regex, a format string, a constant). Names differ; implementations rhyme.
3. **Land on exactly one of three outcomes, and state which in the finding:**

   - **EXISTS, equivalent** → "Use `formatCurrency` from `src/lib/money.ts` instead." SHOULD-FIX. **BLOCKER** when two copies can silently diverge on logic that must agree everywhere — validation, auth, money, dates, permission checks, serialisation.
   - **EXISTS, close but not identical** → "Extend `retry()` in `src/lib/http.ts` with a `backoff` option rather than adding a second implementation." SHOULD-FIX. Say explicitly whether extending it risks existing callers, and if it does, say so instead of pretending it's free.
   - **DOES NOT EXIST and is genuinely general** → "Promote to `src/lib/<name>.ts`." SHOULD-FIX. Only when the function has no local state, no domain coupling, and a plausible second caller. A one-off helper stays where it is — **do not over-abstract; a premature util is its own defect.**

4. **If you could not complete the search** — repo too large, no discoverable shared location, generated code — say so in **Coverage → Not verified**. Never write "this may already exist somewhere" as a finding.

### 3.3 New dependencies

Any third-party package added in this PR:

- Name it explicitly in the review with what it's for. A dep added for something the standard library or an already-present dep does → SHOULD-FIX, name the alternative.
- A package you cannot find on the registry, or whose name is a near-miss of a popular package → **BLOCKER (SECURITY)**. Hallucinated and typosquatted dependencies are a live supply-chain attack, and AI-assisted PRs are where they show up.
- Unpinned or wide version range where the repo otherwise pins → SHOULD-FIX.
- A heavy dep pulled in for one function → SHOULD-FIX (see also the web profile's bundle rules).

### 3.4 Config and secrets discipline

- A new environment variable or config key must (a) appear in `.env.example` / the config schema / the deployment manifest, **and** (b) fail fast at startup with a clear message when missing. Either one absent → SHOULD-FIX.
- A silent default for a **required** secret or credential → **BLOCKER (SECURITY)**. Silent defaults ship to production and nobody notices until it matters.
- Config read at module import time in a way that makes it untestable or unreloadable → NIT unless it breaks something.

### 3.5 PR scope and reviewability

Reviewability is a quality attribute, not a courtesy.

- A PR mixing an unrelated refactor with a behaviour change → PROCESS SHOULD-FIX. **Name the seam** where it should be split.
- More than ~25 files across unrelated areas, or a rename/format sweep bundled with logic → PROCESS SHOULD-FIX with a proposed split.
- Empty or intent-free PR description on a non-trivial change → PROCESS SHOULD-FIX.

### 3.6 Time, money, identifiers

Three categories that produce quiet, expensive bugs across every profile:

- Naive or local-timezone datetimes stored, compared, or serialised; business logic on server-local `now()` → CORRECTNESS SHOULD-FIX (store UTC, convert at the edge). **BLOCKER** when it affects billing periods, scheduling, expiry, or audit records.
- Float or double used for money → **BLOCKER (DATA)**. Integer minor units or a decimal type.
- Sequential or guessable identifiers newly exposed in a public URL or API response → SECURITY SHOULD-FIX (and check the authz scoping — see the service profile's IDOR rule).

### 3.7 Plausible-but-wrong code

A class of defect that passes CI, reads fluently, and survives a skim review. It shows up in any fast-moving codebase and especially in AI-assisted PRs, because the failure mode of generated code is *confident plausibility*, not obvious breakage.

**Two rules on how to report these, and they are not optional:**

1. **Verify before asserting.** Every item below requires a grep or a file read first. "This looks unused" without having searched for callers is not a finding.
2. **Never write "this looks AI-generated" in a review.** Describe the defect. Speculating about authorship is unfalsifiable, insulting, and tells the author nothing about what to change.

**Dead weight**
- New function, export, file, type, or config key with no caller anywhere in the repo → MAINTAINABILITY SHOULD-FIX. Grep the whole repo for the identifier before flagging; something exported for a consumer outside the repo is not dead.
- Unused parameter, unused import, or a variable assigned and never read → NIT (or leave it to the linter if the repo has one that catches it).
- Placeholder residue: `// ... rest of implementation`, `TODO: implement`, an ellipsis comment standing in for a body, or example/sample values left in a real code path → SHOULD-FIX. **BLOCKER** if the placeholder is on a path that executes.

**Invented surface**
- A method, option, or config key used on a third-party library that does not exist in the version this repo has installed → **BLOCKER (CORRECTNESS)**. Verify against the lockfile version and the actual package in `node_modules` / `site-packages` / vendor, **not from memory** — your recollection of a library's API is exactly as unreliable as the author's.
- An internal function called with a signature that does not match its definition, or a field read off a type that does not declare it → **BLOCKER (CORRECTNESS)**.

**Speculative generality**
- An interface, factory, strategy, or base class introduced with exactly one implementation and one call site → MAINTAINABILITY SHOULD-FIX. Say what the concrete version would look like.
- A parameter, option, or feature flag added that every call site passes the default for → SHOULD-FIX.
- A configurable value that is never configured anywhere → NIT.

**Defensive scaffolding that hides real behaviour**
- `try`/`catch` wrapped around code that cannot throw, or a `catch` that logs and continues where the caller needs to know the operation failed → CORRECTNESS SHOULD-FIX. **BLOCKER** if it swallows a failed write or a failed auth check.
- Null/undefined guards on values that the type system or the call path guarantees are present → NIT; the noise makes the *real* guards invisible.
- An unreachable fallback branch, or a default case that can never be hit → NIT, but say which condition makes it unreachable so the author can confirm.

**Drifted or decorative documentation**
- A comment or docstring describing behaviour the function does not have → SHOULD-FIX. A wrong comment is worse than no comment; it will be trusted.
- Comments that restate the line beneath them (`// increment the counter`) → NIT, capped at one consolidated note per PR.

**Stitched-together style**
- Two ways of doing the same thing introduced *within a single PR* — two HTTP clients, two date libraries, two error shapes, two naming conventions → MAINTAINABILITY SHOULD-FIX. Within one PR this is not a legacy-drift excuse.
- A new module that duplicates an existing module's purpose under a different name → SHOULD-FIX; this is §3.2 again, and it is the most common instance of it.

**Tests that don't test**
- A test that would still pass against an empty function body → TESTS SHOULD-FIX; state what it should assert.
- A test asserting only that a mock was called, never on the resulting behaviour or return value → SHOULD-FIX.
- A test whose name describes a case its body does not actually exercise → SHOULD-FIX.

## 4. Verdict rules — mechanical, no judgement

Compute the verdict from the counts. Do not override it with a vibe.

| Condition | Verdict |
|---|---|
| ≥1 BLOCKER | `NO-GO` |
| 0 BLOCKERs, ≥1 SHOULD-FIX | `GO WITH FIXES` |
| 0 BLOCKERs, 0 SHOULD-FIX | `GO` |

One exception, and it must be stated: if you could not verify something material (no access to a dependency, generated code you cannot read, a migration whose data volume you don't know), the verdict is `NEEDS INFO` and you list exactly what you need. Never guess your way to a `GO`.

## 5. Required output template

Emit exactly this structure. No preamble, no closing summary paragraph.

```markdown
## Review — <repo>#<pr-number>: <pr title>

**Verdict:** GO | GO WITH FIXES | NO-GO | NEEDS INFO
**Profile:** web | service | gateway | ai-agent  ·  **Files:** N  ·  **Diff:** +X / −Y
**Intent:** <one line: what this PR is trying to do>

### Blockers (N)
<!-- omit the section body and write "None." if empty -->
**B1 · [CATEGORY] `path/to/file.ts:42`**
What is wrong. What happens if it merges. What to do instead.

### Should fix (N)
**S1 · [CATEGORY] `path/to/file.ts:88`**
What is wrong and the suggested change.

### Nits (N)
- `path/to/file.ts:12` — brief note.

### Coverage
- **Verified:** <what you actually read and reasoned about>
- **Not verified:** <what you could not check — generated files, external services, perf under load, an incomplete utils search>
- **Tests:** <added / updated / none — and whether that is acceptable for this change>
```

`CATEGORY` is one of: `SECURITY`, `CORRECTNESS`, `DATA`, `PERF`, `API-CONTRACT`, `TESTS`, `OBSERVABILITY`, `MAINTAINABILITY`, `PROCESS`.

## 6. Tone

Write for the author, not about them. State the problem and the fix. No praise sandwich, no "great work!", no hedging like "you might want to consider possibly". If a finding is a matter of preference, it is a NIT and should be labelled as one.

## 7. Repo profile resolution

Determine which repo-type skill to layer on, in this order:

1. `.claude/review-profile.yml` in the repo → read its `profile:` key.
2. `CLAUDE.md` containing a line matching `review-profile: <type>`.
3. Auto-detect and **state the guess in the output**:
   - `gateway` — Kong/Envoy/Traefik/APISIX config, an `api-gateway`/`edge`/`proxy` naming convention, route tables with upstream targets, API-key or rate-limit middleware as the repo's main concern.
   - `ai-agent` — prompts/ directory, evals/, LLM SDK deps (anthropic, openai, langchain, llamaindex), tool/function schema definitions.
   - `web` — package.json with react/vue/svelte/next/nuxt/angular, or an index.html entry point.
   - `service` — Dockerfile + server framework (fastapi, express, gin, spring, nest), or an api/ + migrations/ layout.
4. If detection is ambiguous, apply `review-service` and say the profile was uncertain.

A repo may legitimately need two profiles (an agent service behind its own gateway). If the diff clearly spans both, load both and say so in the output — the checks are additive and never conflict.

## 8. Per-repo overrides

`.claude/review-profile.yml` may carry an `overrides:` block (see `review-profile.example.yml` in this skill folder). Honour it as follows:

- `overrides.blockers` — additional repo-specific BLOCKER conditions. Apply them exactly as written.
- `overrides.waivers` — a shared rule this repo has consciously opted out of. Do **not** report it. If you find a waived issue anyway and believe the waiver's stated reason does not hold for this diff, report it as a SHOULD-FIX and quote the waiver.
- `overrides.ignore` — glob paths to skip. List skipped paths in the Coverage "Not verified" line.

Overrides can add strictness or waive a specific named rule. They can never change the severity definitions, the verdict table, or the output template — those are global by design.

The same file may carry two optional hints that make §3 faster and more accurate. Use them when present instead of guessing:

- `shared_code:` — globs for where this repo actually keeps reusable code. Search these first in the §3.2 utils check.
- `constants:` — the modules literals should be moved into, so §3.1 findings can name a real destination path rather than "somewhere in constants".
