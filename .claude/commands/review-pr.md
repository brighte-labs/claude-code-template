# /review-pr — Brighte PR Review Command
# Usage: /review-pr [PR-number or branch name]
# Example: /review-pr 247
# Example: /review-pr feature/BTB-4521-repayment-calculator
# Performs a full quality gate review on an existing PR before human review

---

## Purpose

Run the full Brighte quality gate on a PR that was NOT created by `/feature`
(e.g., created manually, by another tool, or by a team member without Claude Code).
Also useful to re-run the gate on a PR after changes were made in response to reviewer feedback.

Read `.claude/commands/quality-gate.md` now — you will execute it in full.

---

## Step 1 — Orient

```
Read tasks/lessons.md → load all past corrections
Fetch the PR / checkout the branch
Identify all files changed in this PR vs the base branch
```

---

## Step 2 — Context Assessment

Before running the gate, understand what you're reviewing:

- Which Brighte service(s) are touched?
- Is this a feature, bug fix, hotfix, or infrastructure change?
- Identity system in play? (Auth0/Okta / Entra / Neither)
- Does this touch payment data, loan data, customer PII, Finpower/MSSQL?
- Are there any IaC changes (Terraform, GitHub Actions, WAF rules)?
- Is PHP Comms involved?

Use this context to calibrate gate severity — payment/Finpower/PII changes get maximum scrutiny.

---

## Step 3 — Run the Full Quality Gate

Execute `.claude/commands/quality-gate.md` in full on all changed files.

No shortcuts. All waves. Full report.

---

## Step 4 — Generate PR Walkthrough

Run the `pr-summary` subagent. Pass:
- Files changed: `[files identified in Step 1]`
- Ticket: `[from PR title/description if present]`
- Description: `[PR title]`
- Risk tier: `[assessed in Step 2]`
- Gate result: `[composite score + key findings from Step 3]`

Post the walkthrough as a PR comment immediately:
```bash
gh pr comment $PR_NUMBER --body "[walkthrough output]"
```

---

## Step 5 — Output Review Report

After the gate completes, produce a review report suitable for adding as a PR comment:

```markdown
## Claude Code — Automated Review Report

**Quality Gate Result:** [✅ PASS / ⚠️ CONDITIONAL / 🔴 FAIL] — [composite]/100

| Check | Score | Result |
|---|---|---|
| security-reviewer | [X]/100 | [result] |
| code-security | [X]/100 | [result] |
| code-reviewer | — | [result] |
| infra-reviewer | — | [result or N/A] |
| compliance-checker | — | [result] |
| lessons-checker | — | [result] |
| **COMPOSITE** | **[X]/100** | **[result]** |

### Auto-remediated
[List of CRITICAL/HIGH findings fixed automatically, or "None"]

### Requires Human Review
[List of MEDIUM/LOW findings + compliance items needing engineer decision, or "None"]

### WAF Gap Exposure
[YES with list / NO]

### Blocking Issues
[List — or "None — PR approved for human review"]
```

---

## Step 6 — Verdict

**If GATE PASS (≥ 85):** 
→ PR is approved from Claude Code's perspective. Ready for human/Copilot review.

**If CONDITIONAL PASS (70–84):**
→ 👤 Surface MEDIUM findings to engineer. Ask for acknowledgement before marking ready for review.

**If GATE FAIL (< 70):**
→ List all blocking issues clearly.
→ Offer to fix them: "Shall I fix these and re-run the gate?"
→ Do NOT mark PR as ready for review.
