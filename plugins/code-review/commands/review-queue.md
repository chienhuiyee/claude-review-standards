---
description: Sweep every PR waiting on your review across all your repos and produce one consolidated triage table plus a full review per PR.
argument-hint: "[org or owner name, optional]"
allowed-tools: Bash, Read, Grep, Glob, Skill, Agent
---

# Review queue sweep

Owner/org filter: **$ARGUMENTS** (if empty, use PRs where the current user is a requested reviewer, across all repos).

## Step 1 — Find the queue

PRs awaiting my review:

`!gh search prs --review-requested=@me --state=open --json repository,number,title,url,updatedAt,author --limit 40 2>/dev/null || gh api "/search/issues?q=is:pr+is:open+review-requested:@me" --jq '.items[] | {title,url}'`

If the queue is empty, say so and stop.

## Step 2 — Triage table first

Before any deep review, print a table so the user can pick:

| # | Repo | PR | Title | Profile | Size (+/−) | Age |

Profile per repo is resolved with the rules in `code-review:review-core` section 7.

Then ask which to review: all, or a subset. If the user has already said "all" or this is running unattended, proceed with all of them.

## Step 3 — Review each, in parallel

For each selected PR, spawn a **separate `code-review:pr-reviewer` subagent** via the Agent tool so each review has a clean context and none of them bleed into each other. Give each agent: the repo, the PR number, and the resolved profile. Launch them in a single message so they run concurrently.

Each subagent must return the `review-core` output template verbatim.

## Step 4 — Consolidated output

Print, in this order:

1. **Verdict roll-up** — one line per PR:
   `NO-GO  ·  org/repo#123  ·  2 blockers (SECURITY, DATA)  ·  <title>`
   Sorted NO-GO → NEEDS INFO → GO WITH FIXES → GO.
2. **Cross-cutting patterns** — anything that showed up in 2+ PRs (same missing test discipline, same auth gap, same library misuse). This is the section that actually improves the team; do not skip it, and write "None observed" if there are none.
3. **The full per-PR reviews**, unchanged, in roll-up order.

Do not post anything to GitHub in this command. Offer at the end: "Post any of these as PR comments? (list the numbers)".
