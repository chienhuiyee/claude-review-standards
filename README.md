# claude-review-standards

Centralised code review standards for Claude Code, distributed as a plugin marketplace.

One rubric, one output format, four repo-type profiles (`web`, `service`, `gateway`, `ai-agent`). Standards live here; every repo installs the plugin instead of carrying its own copy.

---

## What's in it

```
.claude-plugin/marketplace.json        # makes this repo a marketplace
plugins/code-review/
  .claude-plugin/plugin.json
  commands/
    review-pr.md                       # /code-review:review-pr [n]
    review-queue.md                    # /code-review:review-queue
  agents/
    pr-reviewer.md                     # subagent, one clean context per PR
  skills/
    review-core/SKILL.md               # severity + verdict + universal rules + template
    review-core/review-profile.example.yml
    review-web/SKILL.md                # + i18n discipline
    review-service/SKILL.md
    review-gateway/SKILL.md            # rate limits, API keys, header trust (also loads service)
    review-ai-agent/SKILL.md           # + model agnosticism, prompt storage
```

`review-core` is the piece that fixes inconsistent output. It defines severity by consequence, computes the verdict mechanically from blocker counts, fixes the output template, and carries the **universal engineering rules** that apply in every repo:

| §  | Rule |
|----|------|
| 3.1 | No hardcoded literals — text, status strings, business-rule numbers go to constants/enums/config |
| 3.2 | Reusability — a bounded utils-scan procedure with three defined outcomes (use / extend / promote) |
| 3.3 | New dependencies — justified, pinned, and *actually exists on the registry* |
| 3.4 | Config and secrets — documented, fails fast, never silently defaulted |
| 3.5 | PR scope — refactor mixed with behaviour change gets a split request |
| 3.6 | Time, money, identifiers — UTC storage, no floats for money, no guessable public IDs |
| 3.7 | Plausible-but-wrong code — dead weight, invented library APIs, speculative abstraction, tests that don't test |

Repo-type skills only add *what to look for* — they never change the format or the bar.

---

## Setup — once

1. Push this directory to GitHub as its own repo (private is fine):

   ```bash
   cd claude-review-standards
   git init && git add -A && git commit -m "Centralised code review standards"
   gh repo create <your-org>/claude-review-standards --private --source=. --push
   ```

2. In Claude Code, add the marketplace and install:

   ```
   /plugin marketplace add <your-org>/claude-review-standards
   /plugin install code-review@review-standards
   ```

   Installed at user level, this is available in every repo you open — no per-repo setup needed to start using it.

---

## Setup — per repo (2 minutes each, optional but recommended)

**Pin the profile** so it isn't auto-detected. Create `.claude/review-profile.yml`:

```yaml
profile: service   # web | service | ai-agent
```

**Auto-enable for teammates.** Commit `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "review-standards": {
      "source": {
        "source": "github",
        "repo": "<your-org>/claude-review-standards"
      }
    }
  },
  "enabledPlugins": ["code-review@review-standards"]
}
```

Anyone who clones the repo gets the same reviewer after trusting the folder.

---

## Daily use

**One PR, in the repo:**

```
/code-review:review-pr 482
```

Reads the diff via `gh`, resolves the profile, applies core + type checklist, prints the verdict block, and offers to post it as a PR comment.

**All PRs waiting on you, across repos:**

```
/code-review:review-queue
```

Lists everything where you're a requested reviewer, reviews each in a parallel subagent, then prints a verdict roll-up, cross-cutting patterns, and the full reviews. This is the one that replaces opening five VS Code windows.

---

## The output contract

```
## Review — repo#482: Add bulk export endpoint
**Verdict:** NO-GO
**Profile:** service · **Files:** 7 · **Diff:** +214 / −18
**Intent:** Let admins export all org records as CSV.

### Blockers (1)
**B1 · [SECURITY] `api/exports.py:41`**
Export query filters by `org_id` from the request body, not the session...

### Should fix (2) ...
### Nits (3) ...
### Coverage
- Verified: ... / Not verified: ... / Tests: ...
```

Verdict is mechanical: any blocker → `NO-GO`; no blockers but should-fixes → `GO WITH FIXES`; clean → `GO`; couldn't verify something material → `NEEDS INFO`.

---

## Evolving the standards

When a review misses something real, add the check to the relevant `SKILL.md`, bump the version in `plugin.json`, push. Everyone picks it up on `/plugin marketplace update review-standards`. That's the payoff of centralising: a lesson learned in one repo lands in all of them.

Repo-specific rules go in that repo's `.claude/review-profile.yml` under `overrides:` — see `review-profile.example.yml`. Keep those short; anything general belongs here.
