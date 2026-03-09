# /batch — Brighte Bulk Transformation Command
# Usage: /batch "[transformation rule]" [scope]
#
# Examples:
#   /batch "add set -euo pipefail to every Bash script missing it" scripts/
#   /batch "migrate Python 3.9 type annotations to 3.11 syntax" src/
#   /batch "add Brighte audit log call to all Finance Core endpoints missing it" services/finance-core/
#   /batch "rename ms-identity to user-service across all service files" .
#   /batch "enforce ECR image tags — replace 'latest' with environment-specific tags" terraform/
#
# USE /batch WHEN:
#   - The same transformation applies to 3+ files
#   - Files can be transformed independently (no cross-file logic dependencies)
#   - The change is mechanical and uniform — not creative implementation
#
# USE /feature WHEN:
#   - The change requires planning, design, and sequential implementation
#   - Different files need different logic
#   - The change is a feature or bug fix, not a transformation
#
# USE /simplify WHEN:
#   - You want code quality improvements, not a specific uniform transformation
#
# /batch DOES write files. A PR is created at the end.

---

## Step 1 — Parse arguments

Extract:
- **Transformation rule** — quoted string describing exactly what to change
- **Scope** — directory, glob, or `.` for whole repo

If either is missing: ask the engineer to clarify before proceeding.

---

## Step 2 — Discover affected files

Use the **researcher (Haiku)** subagent — hard cap: 15 tool calls:

```
Find all files in [scope] that need this transformation.
Return:
  - Complete file list with paths
  - For each file: complexity estimate (LOW / MEDIUM / HIGH)
    LOW   = < 50 lines, simple structure, pattern clearly present
    MEDIUM = 50–200 lines, moderate complexity
    HIGH  = > 200 lines, complex structure, or edge cases likely
  - Files with dependencies that affect transformation order
  - Files to exclude (generated, vendor, test fixtures, binary files)
Hard cap: 15 tool calls.
```

Print discovery results:

```
BATCH DISCOVERY
───────────────
Transformation: "[rule]"
Scope: [path]

Files discovered: [N]
  LOW complexity:    [N] files
  MEDIUM complexity: [N] files
  HIGH complexity:   [N] files

Dependency ordering required: [YES — list / NO]

Files excluded: [N]
  [file] — [reason: generated / vendor / already compliant / etc.]

Estimated tokens: ~[N]k
Estimated time:   ~[N] min
```

👤 **ENGINEER CHECKPOINT** — review the file list.
Confirm, exclude any files, or adjust scope.
If discovery finds > 100 files: STOP. Break into sub-batches and confirm with engineer.
Reply "go" to proceed to dry run.

---

## Step 3 — Dry run on one file

Pick the simplest LOW-complexity file that clearly exhibits the pattern.
Apply the transformation in main context (no subagent). Show before/after:

```
DRY RUN — [filename]

BEFORE:
  [relevant snippet — 5–15 lines]

AFTER:
  [transformed snippet — 5–15 lines]

Rule applied: [one-line summary of the pattern]
```

👤 Wait for engineer to reply "confirmed" before fanning out.
If engineer requests adjustments: refine the rule and re-dry-run.

---

## Step 4 — Risk tier assessment

Assess the batch operation against CLAUDE.md tiers:

| Signal | Tier |
|---|---|
| Files touch Finance Core payments, Auth0/Okta, Finpower/MSSQL, PII, Terraform/IAM, WAF, PHP Comms | **Tier 3** |
| Standard application code, endpoints, GitHub Actions, deps | **Tier 2** |
| Docs, Dockerfiles, CI config only, test files, static assets | **Tier 1** |

If ANY file in the batch is Tier 3: the whole batch runs a Tier 3 gate on the final diff.
If a HIGH-complexity file is Tier 3: route it through `/feature` instead — do not batch it.

Declare:
```
🎯 BATCH RISK TIER: [1 / 2 / 3]
   Reason: [what triggers this tier]
   Gate on final diff: Tier [N]
```

---

## Step 5 — Parallel implementation

Group files by complexity. Spawn subagents for LOW and MEDIUM files in parallel.
Implement HIGH-complexity files in main context.

**For LOW-complexity files:**
Spawn `code-transformer` (Sonnet) subagent per file (or per group of 3–5 small files).

```
Use the code-transformer subagent on [file] with:
  Rule: [transformation rule]
  Dry-run example: [before/after from Step 3]
  CLAUDE.md standards: maintain all language-specific conventions
```

**For MEDIUM-complexity files:**
Spawn `code-transformer` (Sonnet) subagent per file, one at a time within the group
(parallel is fine — they write to different files).

**For HIGH-complexity files:**
Implement in main context. Apply the transformation manually, respecting:
- All CLAUDE.md code standards
- The exact pattern from the dry-run
- No unintended side effects

**Dependency ordering:**
If files must be transformed in order (e.g. a type change that affects callers):
transform the dependency first, verify, then proceed to dependents.

---

## Step 6 — Collect results and handle exceptions

After all subagents complete, collect their summaries:

```
IMPLEMENTATION RESULTS
──────────────────────
SUCCESS:  [N] files — transformed cleanly
SKIP:     [N] files — pattern not present (excluded from PR)
PARTIAL:  [N] files — need manual attention (listed below)
FAILED:   [N] files — error during transform (listed below)
```

For each PARTIAL or FAILED file: implement in main context or flag for engineer decision.
If > 20% of files are PARTIAL or FAILED: STOP. Surface to engineer — rule may need refinement.

---

## Step 7 — Run quality gate on the complete diff

Execute `quality-gate.md` for the tier declared in Step 4.
Scope = ALL files transformed in this batch.

The gate runs ONCE on the complete diff — not per-file.
Handle GATE PASS / CONDITIONAL PASS / FAIL exactly as in `feature.md` Step 8.

If gate is Tier 2 or 3: also run `/simplify` on the batch diff (Step 8.5 equivalent)
before creating the PR, if code-reviewer surfaces yellow/green findings.

---

## Step 8 — Create PR

**Branch:** `batch/[transformation-slug]-[N]-files`

**PR title:**
```
[BATCH] [transformation rule — max 60 chars] — [N] files — Tier [N] | Gate: [score]/100
```

**PR body:**
```markdown
## Summary
Automated bulk transformation across [N] files.
Rule: "[transformation rule]"

## Batch Scope
[N] files transformed. [N] files skipped (already compliant). [N] files need manual attention.

## Transformation Pattern
**Before:**
[snippet from dry run]

**After:**
[snippet from dry run]

## Files Changed
[list or summary — group by service/directory]

## Files Skipped
[list + reason — or "None"]

## Files Needing Manual Attention
[list + reason — or "None"]

## Quality Gate Results
| Check | Score | Result |
|---|---|---|
| security-reviewer | [X]/100 | [result] |
| code-reviewer | — | [result] |
| infra-reviewer | — | [result or N/A] |
| compliance-checker | — | [result or N/A] |
| lessons-checker | — | [result] |
| **COMPOSITE** | **[X]/100** | **[GATE RESULT]** |

## Simplification Applied
[changes from /simplify — or "N/A"]

## Auto-Remediated
[CRITICAL/HIGH findings fixed — or "None"]

## Tests
[N] passing

## Rollback
Revert the branch: `git revert --no-commit [first-commit]..[last-commit]`
```

Create branch. Conventional commits. Push. Open PR against `main`. Do NOT merge.

---

## Step 9 — Capture

```
tasks/todo.md    → mark complete, add PR link
tasks/memory.md  → record: "[rule]" batch pattern + which files were affected
tasks/lessons.md → if the transformation revealed recurring technical debt across
                   many files, log it: what the debt was, what rule fixed it
```

Final summary:

```
✅  PR Created: [branch/URL]
🔄  Batch: [N] files transformed | [N] skipped | [N] manual
🎯  Risk Tier: [N]
🔒  Gate: [composite]/100 — [PASS / CONDITIONAL PASS]
🧪  Tests: [N] passing
⏱   ~[N] min | ~[N]k tokens
```

---

## Abort conditions

Stop and surface to engineer immediately if:
- Engineer rejects dry run (rule needs redesign)
- Discovery finds > 100 files (break into sub-batches first)
- > 20% of files return PARTIAL or FAILED
- A HIGH-complexity Tier 3 file is encountered (route to `/feature`)
- Gate composite < 70 and cannot be auto-remediated
- compliance-checker returns FAIL
- Any CRITICAL security finding that requires architectural decision
