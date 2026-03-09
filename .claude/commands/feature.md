# /feature — Brighte Feature Development Command
# Usage: /feature [TICKET-ID] "[description]"
# Example: /feature BTB-4521 "Add repayment calculator endpoint to Finance Core"
# This command is the ONLY path to creating a feature PR. The quality gate is mandatory.

---

## On Receiving This Command

You are taking ownership of this feature end to end. The engineer typed one command.
Your job is to deliver a production-ready, gate-passed PR with minimal interruptions.
The only times you stop and ask are explicitly marked 👤 below.

Read `.claude/commands/quality-gate.md` now — it defines the gate you will execute at Step 8.

---

## Step 1 — Orient (never skip)

```
Read tasks/todo.md          → pick up any in-progress work first
Read tasks/lessons.md       → load all past corrections, apply immediately
Read tasks/memory.md        → load known file paths, architectural decisions, gotchas
```

If tasks/memory.md does not exist, create it now as an empty file.
If tasks/todo.md references an incomplete task, confirm with the engineer before starting new work.

---

## Step 2 — Risk Tier Assessment (do this BEFORE spinning up any subagents)

Assess the risk tier based on the description and files likely to be touched.
This determines which agents fire — saving tokens and time on low-risk changes.

### Tier 1 — Low risk (~35k tokens, ~4 min)
Change touches ONLY:
Dockerfile, docker-compose, README, docs, CSS/static assets, test files,
CI config (no secret changes), non-sensitive config files.
Does NOT touch: application logic, auth, payments, PII, IaC, secrets.

### Tier 2 — Medium risk (~70k tokens, ~8 min)
Change touches: standard application code, new API endpoints, UI components,
non-payment service logic, dependency updates (package.json, requirements.txt),
GitHub Actions (no secret changes), logging improvements.
Does NOT touch: Auth0/Okta, Finance Core payment logic, Finpower/MSSQL,
customer PII, Terraform/WAF/IAM, PHP Comms.

### Tier 3 — High risk (~120k tokens, ~14 min)
Change touches ANY of:
Finance Core payment/loan logic, Auth0/Okta config, Finpower/MSSQL queries,
customer PII, Terraform/CDK/CloudFormation, IAM roles/policies, WAF rules,
security group rules, VPC config, PHP Comms service, EventBridge schema changes
affecting payment events, secrets rotation, Entra/Conditional Access config.

### Rules
- If ambiguous between Tier 2 and 3 → always escalate to Tier 3
- Tier can be upgraded mid-implementation if files touched turn out to be higher risk

### Declare the tier before any subagents fire

Print:
```
🎯 RISK TIER: [1 / 2 / 3]
   Reason: [one line — what triggers this tier]
   Est. gate time: [4 / 8 / 14] min
   Est. tokens: ~[35k / 70k / 120k]
   Agents that will run: [list]
```

---

## Step 3 — Research + Pre-check (scoped to tier)

### Tier 1 — Researcher only
**researcher (Haiku) — hard cap: 8 tool calls**
```
Targeted search only — do NOT explore the full codebase.
Find: the specific files to change and the existing pattern to follow.
Return in under 8 tool calls.
```
No compliance pre-check for Tier 1.

### Tier 2 — Researcher + lightweight compliance (PARALLEL)
**researcher (Haiku) — hard cap: 10 tool calls**
```
Find: existing patterns, key file paths, auth middleware location,
test patterns for this service. Targeted, not exploratory.
```
**compliance-checker (Opus) — abbreviated pre-assessment**
```
Brief assessment only:
- Which services touched, identity system in play
- Any unexpected regulated data exposure?
- SOC 2 controls affected
- Flag MUST-INCLUDE requirements
Skip deep APRA assessment unless regulated data is detected.
```

### Tier 3 — Researcher + full compliance (PARALLEL)
**researcher (Haiku) — hard cap: 12 tool calls**
```
Find: existing patterns, Auth0/Okta JWT middleware, EventBridge schemas,
test patterns, technical debt in this area. Targeted exploration.
```
**compliance-checker (Opus) — full pre-assessment**
```
Full APRA CPS 234 + SOC 2 pre-assessment:
- Services touched, identity system (Auth0/Okta vs Entra — critical distinction)
- Payment/PII data flows and residency
- APRA requirements to build in from day one (audit logging, encryption)
- WAF gap relevance — if Finpower/MSSQL: WAF is COUNT-only, app layer is only defence
- Breach notification risk level
- Specific compliance controls the implementation MUST include
Return: full pre-assessment checklist
```

---

## Step 4 — Write Plan to todo.md

👤 **ENGINEER CHECKPOINT — review before proceeding**

```markdown
## Feature: [TICKET-ID] [description]
## Date: [today]
## Risk Tier: [1 / 2 / 3] — [reason]
## Service: [which Brighte service]
## Identity system: [Auth0/Okta / Entra / Neither]
## Finpower involved: [YES — WAF COUNT-only, app layer only / NO]
## Compliance requirements: [from pre-check, or "N/A — Tier 1"]

## Implementation steps:
- [ ] [step 1]
- [ ] [step 2]
...

## Acceptance criteria:
- [criterion 1]
...

## Rollback procedure:
[how to undo safely]
```

Print the plan and WAIT for engineer confirmation before writing any code.
If engineer says "go" or gives no objection, proceed.

---

## Step 5 — Write Tests First (TDD, scoped to tier)

**Tier 1:** Basic smoke tests — confirm the change works as intended. Minimal.
**Tier 2:** Unit tests for business logic + edge cases. Integration tests if new endpoints added.
**Tier 3:** Full suite — unit, integration, auth failure, EventBridge payload validation,
boundary conditions. Tests MUST FAIL before implementation (true TDD).

Use test patterns from Step 3 research.

---

## Step 6 — Implement

Work through plan checkboxes autonomously.

**Enforce at all times:**
- Secrets → AWS Secrets Manager or Azure Key Vault. Never hardcoded.
- Azure automation → Managed Identity. AWS → IAM roles. No long-lived keys.
- Auth0/Okta JWT → validate `alg`, `aud`, `iss` on every protected endpoint
- Finpower/MSSQL → parameterise ALL queries, zero string concatenation
- Finance Core → audit log on every sensitive operation (APRA requirement)
- EventBridge → validate event source and schema before processing payload
- PHP Comms → no `eval()`, `exec()`, variable file includes
- Data residency → ap-southeast-2 (AWS) or Australia East (Azure) only
- Git → conventional commit messages, no direct push to main

Run relevant tests after each logical chunk. Update todo.md checkboxes as you go.

**Re-assess tier if needed:**
If implementation reveals the change is touching higher-risk files than expected,
upgrade the tier immediately and declare it:
```
⬆️  TIER UPGRADED: [2→3]
Reason: [what was discovered]
Gate agents updated accordingly.
```

---

## Step 7 — Verify Tests Pass

Run the full test suite for the affected service.
All tests must be GREEN before the quality gate runs.
If tests cannot pass after 3 attempts: STOP, surface the blocker to the engineer.

---

## Step 8 — MANDATORY QUALITY GATE (tiered)

**Non-negotiable. No PR created without this.**

Pass the tier from Step 2 to `quality-gate.md` — it determines which agents run.
Execute `.claude/commands/quality-gate.md` in full for the declared tier.

**GATE FAIL (composite < 70):**
- Do NOT create PR
- Fix all blocking issues autonomously
- Re-run gate from Wave 1
- Proceed only when PASS or CONDITIONAL PASS

**CONDITIONAL PASS (70–84):**
- 👤 Surface MEDIUM findings clearly
- Ask: "Review these — reply 'acknowledged' to proceed or give instructions"
- Wait for engineer response

**GATE PASS (≥ 85):**
- Proceed directly to Step 8.5

---

## Step 8.5 — SIMPLIFICATION PASS (conditional — post-gate only)

Run ONLY when gate result is PASS or CONDITIONAL PASS.

**IF** code-reviewer returned any 🟡 Should Fix or 🟢 Consider items:
- Execute `/simplify` — pass it the changed file list and code-reviewer findings
- After `/simplify` completes:
  - Re-run the test suite — must still be GREEN (if a test breaks, `/simplify` auto-reverts that file)
  - Append the simplification report to the gate report under "Simplification Applied"
- Proceed to Step 9

**ELSE** (code-reviewer returned APPROVE with no yellow/green findings):
- Skip. Record in gate report: `Simplify: N/A — no yellow/green findings`
- Proceed to Step 9

---

## Step 8.7 — PR WALKTHROUGH (always runs after gate PASS or CONDITIONAL PASS)

Run the `pr-summary` subagent now. Pass:
- Files changed: `[FILES_CHANGED from quality-gate.md discovery]`
- Ticket: `[TICKET-ID]`
- Description: `[feature description]`
- Risk tier: `[declared tier]`
- Gate result: `[composite score + key findings]`

The subagent produces a formatted markdown walkthrough with Mermaid diagram.
Store the output — it will be posted as a PR comment in Step 9 and appended to the PR body.

Do NOT post it yet. Proceed to Step 9.

---

## Step 9 — Create PR

**PR title:**
```
[TICKET-ID] [description] — Tier [N] | Gate: [score]/100
```

**PR description:**
```markdown
## Summary
[What this PR does — 2-3 sentences]

## Ticket
[TICKET-ID]

## Risk Tier
Tier [N] — [brief reason]
Gate time: ~[N] min | Tokens: ~[N]k

## Changes
[Files changed with brief description of each]

## Quality Gate Results
| Check | Score | Result | Runs on Tier |
|---|---|---|---|
| security-reviewer | [X]/100 | [result] | 1, 2, 3 |
| code-security (deep scan) | [X]/100 | [result] | 3 only |
| code-reviewer | — | [result] | 1, 2, 3 |
| infra-reviewer | — | [result or N/A] | 2, 3 (IaC only) |
| compliance-checker | — | [result or N/A] | 2, 3 |
| lessons-checker | — | [result] | 1, 2, 3 |
| **COMPOSITE** | **[X]/100** | **[GATE RESULT]** | |

## Simplification Applied
[changes from /simplify — or "N/A — no yellow/green findings"]

## Auto-Remediated
[CRITICAL/HIGH findings fixed by Claude Code — or "None"]

## WAF Gap Exposure
[YES — list unmitigated findings / NO / N/A — Tier 1]

## Compliance Notes
[APRA/SOC2 items — or "N/A — Tier 1"]

## Engineer Action Items
[MEDIUM findings needing human judgement — or "None"]

## Tests
[N] tests passing

## Rollback
[How to revert safely]

## For GitHub Copilot / Human Reviewer
Risk Tier [N] gate ran automatically before this PR was created.
CRITICAL and HIGH findings were auto-remediated by Claude Code.
Items in "Engineer Action Items" above require human judgement.
Tier [N] means [low/medium/high]-risk change — review depth calibrated accordingly.
```

Create branch: `feature/[TICKET-ID]-[slug]`
Conventional commits. Push. Open PR against `main`. Do NOT merge.

Then post the PR walkthrough from Step 8.7 as a PR comment:
```bash
gh pr comment --body "[walkthrough output from pr-summary subagent]"
```

---

## Step 10 — Capture and Update Memory

```
tasks/todo.md   → mark complete, add PR link
tasks/memory.md → add architectural facts, key paths, non-obvious patterns discovered
tasks/lessons.md → log correction patterns or reinforced lessons from this session
```

---

## Step 11 — Final Summary

```
✅ PR Created: [branch/URL]
📋 Ticket: [TICKET-ID]
🎯 Risk Tier: [N] — [reason]
🔒 Gate: [composite]/100 — [PASS / CONDITIONAL PASS]
🔧 Simplify: [N changes applied / N/A — no yellow/green findings]
🧪 Tests: [N] passing
📁 Files: [N] changed
⏱  ~[N] min | ~[N]k tokens

Next:
1. GitHub Copilot agent reviews PR automatically
2. Human reviewer per your team process
3. [Engineer action items if any — or "None required"]
```

---

## Abort Conditions

Stop and surface to engineer immediately if:
- Gate composite < 70 and cannot be resolved autonomously
- CRITICAL finding requires architectural decision (not just a code fix)
- compliance-checker returns FAIL (not conditional)
- Tests cannot pass after 3 attempts
- A CLAUDE.md hard stop is triggered
- Tier is genuinely ambiguous and scope needs engineer clarification
