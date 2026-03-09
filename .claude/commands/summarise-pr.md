# /summarise-pr — PR Walkthrough + Architectural Diagram
# Usage: /summarise-pr [PR-number or branch]
# Example: /summarise-pr 247
# Example: /summarise-pr feature/BTB-4521-repayment-calculator
# Example: /summarise-pr  (no arg = current branch)
#
# Generates a human-readable PR walkthrough with Mermaid architectural diagram
# and posts it as a GitHub PR comment. Designed to give human reviewers the
# context that CodeRabbit's PR summary feature provides — but with full
# Brighte architectural and compliance context baked in.
#
# Can be run:
#   - Standalone on any PR/branch at any time
#   - Automatically by /feature (Step 9.5) and /review-pr (Step 4.5) after gate passes

---

## Step 1 — Resolve the PR context

```bash
# If a PR number was given
PR_ARG="$ARGUMENTS"

# Determine branch and changed files
CURRENT_BRANCH=$(git branch --show-current)
BASE_BRANCH="main"

# Get the file list (same discovery pattern as quality-gate.md)
FILES_CHANGED=$(git diff --name-only ${BASE_BRANCH}...HEAD)
COMMIT_COUNT=$(git rev-list --count ${BASE_BRANCH}...HEAD)
FIRST_COMMIT_MSG=$(git log --oneline ${BASE_BRANCH}...HEAD | tail -1)
LAST_COMMIT_MSG=$(git log --oneline -1)

echo "Branch: $CURRENT_BRANCH"
echo "Commits: $COMMIT_COUNT"
echo "Files changed: $(echo "$FILES_CHANGED" | wc -l)"
echo "$FILES_CHANGED"
```

If `FILES_CHANGED` is empty (branch is behind main or has no commits): stop and tell the engineer.

Read `tasks/todo.md` to get the ticket ID and description if not passed as an argument.

---

## Step 2 — Gather gate context

Look for the gate report in one of these places (in order):
1. Most recent output in this session (if /feature or /review-pr just ran)
2. `tasks/todo.md` — gate score may be noted there
3. Most recent GitHub PR comment (if PR number was given): `gh pr view $PR_ARG --comments`

If no gate context is available, note `Gate: not run` in the output — do NOT run the gate.
This command generates the walkthrough only. Gate is /feature's or /review-pr's responsibility.

---

## Step 3 — Run pr-summary subagent

Pass the following context to the `pr-summary` subagent:

```
Files changed: [FILES_CHANGED list]
Ticket: [ticket ID from todo.md or ARGUMENTS, or "none"]
Description: [from todo.md or ARGUMENTS]
Risk tier: [from gate context, or "not assessed"]
Gate result: [composite score + key findings summary, or "not run"]
Branch: [CURRENT_BRANCH]
Commit count: [COMMIT_COUNT]
```

The subagent will read the diff, build the diagram, and return the formatted markdown walkthrough.

---

## Step 4 — Post as PR comment

**If a PR number was provided OR a PR exists for this branch:**

```bash
# Check if PR exists for current branch
gh pr view --json number,url 2>/dev/null || echo "no-pr"
```

If a PR exists:
```bash
gh pr comment $PR_NUMBER --body "$(cat <<'COMMENT'
[walkthrough output from pr-summary subagent]
COMMENT
)"
echo "✅ Walkthrough posted to PR #$PR_NUMBER"
```

If no PR exists yet:
- Print the walkthrough to the terminal
- Tell the engineer: "No PR found for this branch. Walkthrough printed above — it will be posted automatically when /feature creates the PR, or you can post it manually."

---

## Step 5 — Summary

```
✅  PR Walkthrough complete
📋  Branch: [branch]
📊  Diagram: [included / omitted — reason]
💬  Comment: [posted to PR #N / printed to terminal]
```

---

## When this command is called by /feature or /review-pr

Both commands pass context directly — skip Steps 1 and 2, go straight to Step 3.
After Step 4, return control to the calling command (do not print a separate summary).
