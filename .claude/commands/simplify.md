# /simplify — Brighte Code Simplification Command
# Usage: /simplify [path]
#   /simplify                    → scope = git diff --name-only HEAD (all changed files)
#   /simplify src/payments.py    → scope = that file
#   /simplify src/services/      → scope = all files in that directory
#
# PURPOSE: Fix the code inefficiencies and smells that the quality gate FOUND but did
# NOT auto-remediate — specifically code-reviewer "Should Fix" (🟡) and "Consider" (🟢)
# items, CLAUDE.md non-compliance, and over-engineering patterns.
#
# WHEN TO RUN:
#   - Automatically: called by feature.md Step 8.5 after gate PASS/CONDITIONAL PASS
#     when code-reviewer returned yellow/green findings
#   - Manually: /simplify [path] at any time to clean up a file or module
#
# THIS COMMAND WRITES FILES. It is not read-only.
# After applying changes, it re-runs tests to confirm nothing broke.

---

## Step 1 — Determine scope and input source

**If called from feature.md Step 8.5:**
- Scope = files changed in this session (`git diff --name-only HEAD`)
- Input = code-reviewer findings passed from the gate (yellow + green items)
- Auto-proceed after plan (no engineer checkpoint needed — gate already approved)

**If called standalone with a path:**
- Scope = `$ARGUMENTS`
- Must run own analysis (Step 2B below)
- Wait for engineer confirmation at Step 3

**If called standalone with no argument:**
- Scope = `git diff --name-only HEAD`
- Must run own analysis (Step 2B below)
- Wait for engineer confirmation at Step 3

---

## Step 2 — Load findings or run analysis

### 2A — If findings were passed in (Step 8.5 invocation)

Parse the code-reviewer output from the gate. Collect:
- All 🟡 **Should Fix** items (file + line + description)
- All 🟢 **Consider** items (file + line + description)
- Any 🔴 **Must Fix** items not yet resolved (should not exist post-gate — flag if found)

Skip 2B. Proceed directly to Step 3.

### 2B — If standalone invocation (no gate findings available)

Analyse each file in scope against CLAUDE.md code standards:

**Universal checks (all languages):**
- Dead code: unreachable blocks, unused imports, commented-out code blocks
- DRY violations: logic duplicated 2+ times that could be extracted cleanly
- Complexity: functions > 40 lines or with nesting depth > 4
- Naming: variables/functions with unclear names (single letters, `temp`, `data`, `stuff`)
- Comments: are they explaining *why* (good) or *what* (redundant noise)?
- Over-engineering: abstractions with only one call site, unnecessary generality

**Python-specific:**
- Type hints present on all function signatures?
- Error handling: bare `except:` or `except Exception:` without logging?
- `logging` used correctly — are messages meaningful enough to debug production?
- Idempotency: scripts safe to run twice?

**Bash-specific:**
- `set -euo pipefail` at top?
- Meaningful log context in echo/print statements?
- Temp file cleanup on exit?

**PowerShell-specific:**
- `#Requires -Version 7` present?
- Structured error handling (`try/catch` with typed exceptions)?

**Scripts for Philippines team:**
- Inline English comments explaining intent (not just what — but why)?

---

## Step 3 — Declare simplification plan

Print the plan before touching a single file:

```
SIMPLIFICATION PLAN
───────────────────
Scope: [N] files
Source: [gate findings / standalone analysis]

Findings to address:
  🟡 [file:line] — [description]
  🟢 [file:line] — [description]
  ...

Planned changes:
  [file] — [what will change and why — one line per file]
  [file] — [what will change and why]

Changes NOT planned (require architectural decision):
  [item — or "None"]
```

**If called from Step 8.5:** auto-proceed.
**If called standalone:** wait for engineer to reply "go" before making any changes.

---

## Step 4 — Apply changes

Work through each finding. Apply the fix directly.

**What you MAY do:**
- Extract duplicated logic into a well-named helper (only if called 2+ times)
- Rename unclear variables/functions to descriptive names
- Add missing `set -euo pipefail`, type hints, `#Requires -Version 7`
- Add missing error handling boilerplate (logging, structured exceptions)
- Add missing CLAUDE.md-required patterns (audit log calls, idempotency guards)
- Remove dead code (unused imports, unreachable blocks, commented-out code)
- Split functions > 40 lines into named sub-functions if the split is natural
- Add inline comments where complex logic now exists without explanation
- Improve an existing comment that explains *what* (not *why*)

**What you must NOT do:**
- Change public API signatures, function contracts, or module exports
- Remove logic — only restructure, rename, or clarify
- Add new business logic or features
- Add new tests (that belongs in test-writer)
- Change any security-critical code (secrets, auth, encryption) — flag it instead
- Make more than one conceptual change per function

Commit each changed file:
```
git add [file]
git commit -m "refactor: simplify [what] — [file] (post-gate cleanup)"
```

---

## Step 5 — Verify

After all changes are applied:

1. Run the test suite for affected services — must be GREEN.
   - If a test fails: revert that file's simplification changes (`git checkout HEAD~1 -- [file]`)
   - Note any reverts in the report

2. Inline sense-check: re-read each changed file.
   Ask: "Would a staff engineer at a regulated fintech approve this?"
   If not → revert and flag.

---

## Step 6 — Report

```
SIMPLIFICATION REPORT
─────────────────────
Files changed:    [N]  (of [N] in scope)
Files reverted:   [N]  (tests broke — listed below)
Findings resolved: [N]
Tests:            [status] — [N] passing

Changes applied:
  [file] — [what changed] — [why (finding reference)]
  [file] — [what changed] — [why (finding reference)]

Files reverted (test failures):
  [file] — [which test broke] — needs manual attention
  [or "None"]

Items NOT addressed (require architectural decision):
  [item and reason — or "None"]
```

**If called from feature.md Step 8.5:**
Append this report to the gate report under "Simplification Applied."
Return control to feature.md Step 9 — PR creation.

**If called standalone:**
Done. Run `/review-pr` or create PR via `/feature` if this is part of a PR workflow.
