# /setup — Onboard a New Repo to Brighte Claude Code
# Usage: /setup [target-repo-path]
# Example: /setup .
# Example: /setup ~/projects/finance-core
# Example: /setup  (no arg = current repo)
#
# Run this once when starting Claude Code in any new Brighte repo.
# Installs agents, commands, hooks, and seeds the tasks/ scaffold.
#
# Two modes:
#   Via script (recommended for first install):
#     bash scripts/setup-repo.sh [target]
#
#   Via Claude Code (this command):
#     /setup [target]  — Claude runs the script and then guides you through
#                        the repo-specific memory.md initialisation

---

## Step 1 — Run the setup script

```bash
TARGET="${ARGUMENTS:-.}"
bash scripts/setup-repo.sh "$TARGET"
```

If the script doesn't exist yet (this is the very first install on a brand-new machine):
tell the engineer they need to clone the template repo first, or copy `scripts/setup-repo.sh`
manually. The script is the source of truth for what gets installed.

If the script exits with an error: surface the error and stop. Do not proceed until the install is clean.

---

## Step 2 — Initialise tasks/memory.md for this repo

After a successful install, memory.md exists but is empty.
Ask the engineer the following questions to seed it (or infer from the repo if obvious):

```
📋 Let's seed memory.md with repo-specific context so Claude Code knows this codebase from session one.

Answer what you know — skip anything unclear:

1. Service name: What does this repo do in one sentence?
   (e.g. "Finance Core — lending logic and core financial processing")

2. Brighte account: Which AWS account does this deploy to?
   (e.g. Prod General 590257951365 / Prod Finpower 114211845990 / Labs / etc.)

3. Key entry point: What's the main file or entry point?
   (e.g. "src/index.ts", "app.py", "cmd/main.go")

4. Identity system: Does this service touch Auth0/Okta, Entra ID, both, or neither?

5. Sensitive paths: Any paths that touch payments, PII, or Finpower/MSSQL?
   (e.g. "src/payments/ → Fatzebra integration")

6. Non-obvious gotchas: Anything about this repo that will surprise Claude?
   (e.g. "Uses a custom auth middleware that wraps Auth0 tokens", "Port 4000 hardcoded in Dockerfile")

7. Team: Who owns this? (team name for tagging)
```

Wait for the engineer's answers. Then write them to `tasks/memory.md`:

```markdown
# [Service Name] — Claude Code Memory
# Seeded by /setup on [date]

## Service
[one-sentence description]

## AWS Account
[account name + ID]

## Key Paths
- Entry point: [path]
- Sensitive paths: [list or "none identified yet"]

## Identity System
[Auth0/Okta / Entra / Both / Neither]

## Team
[team name]

## Non-obvious gotchas
[list or "none yet — add as discovered"]

## Architectural decisions
[empty — add as discovered during /feature sessions]
```

---

## Step 3 — Verify the /start session

Run `/start` to confirm everything loads correctly:
- lessons.md Brighte gotchas loaded
- memory.md populated
- todo.md clean
- Session declaration prints cleanly

---

## Step 4 — Final checklist for the engineer

Print this checklist:

```
✅  Claude Code is ready for: [repo path]

Before your first PR from this repo, verify:

□  CLAUDE.md infrastructure context is accurate
   (AWS account IDs, service name, VPC details if different from template)

□  Wiz action SHAs are pinned in your GitHub Actions workflows
   Template uses wizcodetest/github-action-iac-scan@v1 and
   wizcodetest/github-action-container-scan@v1 — replace @v1 with commit SHAs
   from: https://github.com/wizcodetest

□  GitHub environment protection rules exist for prod
   (Settings → Environments → prod → Required reviewers)

□  tasks/ is in .gitignore
   (setup-repo.sh does this automatically — verify it was applied)

□  WIZ_CLIENT_ID and WIZ_CLIENT_SECRET are in GitHub Secrets for this repo
   (Settings → Secrets → Actions)

Ready. Type /start to begin your first session.
```
