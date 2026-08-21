---
description: Standardised review of a GitHub PR against this repo's profile. Usage: /review-pr [PR number or URL]
argument-hint: "[pr-number|url] (defaults to the PR for the current branch)"
allowed-tools: Bash, Read, Grep, Glob, Skill
---

# Standardised PR review

Target PR: **$ARGUMENTS** — if empty, use the PR for the current branch (`gh pr view --json number`).

Follow this sequence exactly. Do not improvise the format.

## Step 1 — Load the contract

Invoke the `code-review:review-core` skill first and follow it. It defines severity, the verdict rules, and the output template. Nothing below overrides it.

## Step 2 — Resolve the repo profile

Repo profile config (empty means none is set):

`!cat .claude/review-profile.yml 2>/dev/null || grep -i "review-profile" CLAUDE.md 2>/dev/null || echo "NO PROFILE SET"`

Resolve using the rules in section 7 of `review-core`, then load the matching profile skill:

- `code-review:review-web`
- `code-review:review-service`
- `code-review:review-gateway` — **also load `review-service`**, a gateway is still a service
- `code-review:review-ai-agent`

If the diff clearly spans two profiles (an agent service that also owns its edge routing), load both and say so.

If the profile came from auto-detection rather than config, say so in the output and, at the end, suggest the one-line `.claude/review-profile.yml` that would pin it.

## Step 3 — Gather the PR

Run these (substitute the resolved PR number):

- `gh pr view <n> --json number,title,body,author,baseRefName,headRefName,additions,deletions,changedFiles,files`
- `gh pr diff <n>`
- `gh pr checks <n>` — if CI is failing, note it; a failing build is a `PROCESS` blocker unless the failure is unrelated and stated.

Read surrounding files with Read/Grep wherever the diff alone is not enough to judge correctness. **Do not review from the diff alone when a finding depends on context you have not read** — either read it, or move the finding to the Coverage "Not verified" list.

## Step 4 — Review

Walk the repo-type checklist against the diff. For each candidate finding, before writing it down, confirm:

1. You can point to a specific `file:line` in the diff.
2. You can state the concrete consequence if merged.
3. The severity follows the definitions in `review-core`, not your mood.

Discard anything that fails those three checks. A short honest review beats a padded one.

## Step 5 — Output

Emit the `review-core` template and nothing else. Compute the verdict mechanically from the counts.

Then, and only then, ask the user (one line): **"Post this as a PR comment?"** If they say yes, run:

```
gh pr comment <n> --body-file <file>
```

Do not post without confirmation.
