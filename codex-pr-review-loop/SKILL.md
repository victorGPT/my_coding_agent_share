---
name: codex-pr-review-loop
description: "Automate the post-PR review loop: after a PR is submitted, poll GitHub review threads every 5 minutes, wait for new updates, fix issues from updated review threads, commit and push, then begin the waiting loop again, reply with @codex review as a separate PR issue comment, and continue until a Codex review comment states 'Didn't find any major issues', then merge to remote main and pull locally. Use when asked to manage the ongoing review/iteration cycle for a PR." 
---

# Codex PR Review Loop

## Overview
Run a 5-minute polling loop for PR review threads, act only when new review updates appear, fix issues found, commit+push, post a standalone `@codex review` PR comment, and repeat until Codex reports no major issues, then merge to remote `main` and pull locally.

## Workflow (Loop)

### Preconditions
1. PR already exists.
2. `gh` is authenticated.
3. You are allowed to push/merge as part of this explicit user-requested workflow.

### Step 0: Identify PR
- Prefer current branch PR: `gh pr view --json number`.
- If ambiguous, ask for PR number.

### Step 1: Poll review threads and PR issue comments
Run:
```
GH_OWNER=<owner>
GH_REPO=<repo>
PR_NUMBER=<number>

gh api graphql -f owner="$GH_OWNER" -f name="$GH_REPO" -F number=$PR_NUMBER -f query='query($owner:String!,$name:String!,$number:Int!){repository(owner:$owner,name:$name){pullRequest(number:$number){reviewThreads(first:100){nodes{isResolved path line originalLine comments(first:50){nodes{author{login} body createdAt reactions(content: THUMBS_UP){totalCount}}}}} comments(first:100){nodes{author{login} body createdAt reactions(content: THUMBS_UP){totalCount}}}}}}}'
```

### Step 2: Wait for updates (repeat until update)
- **Do not proceed** unless review threads **or PR issue comments** have **new updates** since the last poll (new comments, new threads, or updated timestamps).
- If no updates are found, **wait 5 minutes and poll again**. Continue doing this automatically until updates appear.

### Step 3: Decide
- **If any review thread comment OR PR issue comment contains** `Codex Review: Didn't find any major issues` **(exact phrase)** **OR** a thumbs-up signal:
  - thumbs-up signal = comment body includes `👍` or `:+1:` **OR** `reactions(content: THUMBS_UP).totalCount > 0`
  - You may merge to remote `main`, then pull locally. Proceed to Step 6.
- **Else:** treat updated review thread feedback as actionable. Proceed to Step 4.

### Step 4: Fix bugs or requested changes
- Apply fixes in code.
- Add/adjust tests for regressions.
- Run relevant verification commands.
- Commit changes locally.
- Push to update the PR.

### Step 5: Reply and wait
- Post a **separate PR issue comment** (not a thread reply) with **exactly** `@codex review`.
  - Use: `gh pr comment <PR_NUMBER> --body "@codex review"`
- After replying, immediately return to Step 2 (wait 5 minutes, poll again until updates appear).

### Step 6: Merge and sync
- Merge PR to `main` on remote (e.g., `gh pr merge --merge` or project-required strategy).
- Pull latest `main` locally.

## Guardrails
- Do not merge until review thread content **or PR issue comments** explicitly include: `Codex Review: Didn't find any major issues` **or** a thumbs-up signal (see Step 3).
- Do not take any action (fix/respond/merge) unless review threads have new updates since the last poll.
- If no updates are found, you MUST keep waiting in 5-minute increments until updates appear.
- Always run tests appropriate to the change before pushing.
- Follow repo rules (e.g., no `git push` unless this skill is explicitly requested).
- If merging/pushing requires confirmation in the current environment, ask for explicit confirmation.
