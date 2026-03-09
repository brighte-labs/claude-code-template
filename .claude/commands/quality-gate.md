# Brighte Mandatory Quality Gate — Tiered
# Invoked by: feature.md, review-pr.md, incident.md
# Caller must declare the risk tier before invoking this gate.
# DO NOT skip, shortcut, or reorder steps within the declared tier.
# NO PR is created until this gate returns PASS or CONDITIONAL PASS.

---

## Tier Reference

| Tier | Risk Level | Est. Tokens | Est. Time | Typical change |
|---|---|---|---|---|
| 1 | Low | ~35k | ~4 min | Dockerfile, docs, CSS, test files |
| 2 | Medium | ~70k | ~8 min | App code, new endpoints, deps |
| 3 | High | ~120k | ~14 min | Payments, auth, Finpower, IaC, PII |

If tier was not declared by the calling command, assess it now using the rules in `feature.md` Step 2.
When in doubt, escalate to Tier 3.

---

## WAVE 1 — File Discovery (run before launching any agents)

Resolve the file list once here so agents receive an explicit list — not a vague scope description.
This prevents agents from running git commands themselves and avoids repeated permission prompts.

```bash
FILES_CHANGED=$(git diff --name-only HEAD)
IAC_FILES=$(echo "$FILES_CHANGED" | grep -E "\.(tf|tfvars)$|\.github/workflows/|task-definition|appspec|Dockerfile")
```

- Pass `FILES_CHANGED` to: code-reviewer, security-reviewer, code-security
- Pass `IAC_FILES` to: infra-reviewer (skip infra-reviewer entirely if `IAC_FILES` is empty)
- If `FILES_CHANGED` is empty: stop and surface to engineer — nothing to review

---

## WAVE 1 — Run per tier (all parallel within the tier)

### TIER 1 — Wave 1: 2 agents

**1A. security-reviewer (Opus)**
```
Files to review: [FILES_CHANGED from discovery above]
Run the standard checklist. Focus on: secrets, obvious injection risks,
hardcoded credentials, insecure patterns.
Return: findings + 0-100 score.
Cap tool calls at 10.
```

**1B. code-reviewer (Sonnet)**
```
Files to review: [FILES_CHANGED from discovery above]
Focus: correctness, edge cases, Brighte code standards, lessons.md patterns.
Return: APPROVE / REQUEST CHANGES with line references.
Mark "Should Fix" (🟡) and "Consider" (🟢) items explicitly as /simplify candidates
— these are consumed by feature.md Step 8.5, not auto-remediated by the gate.
Cap tool calls at 10.
```

Skip: code-security, infra-reviewer, compliance-checker for Tier 1.

---

### TIER 2 — Wave 1: 3 agents (+ conditional 4th)

**1A. security-reviewer (Opus)**
```
Files to review: [FILES_CHANGED from discovery above]
Full checklist including: Auth0/Okta JWT patterns (if auth touched),
injection risks, secrets, SSRF, input validation.
Return: findings + 0-100 score + WAF gap exposure.
Cap tool calls at 15.
```

**1B. code-reviewer (Sonnet)**
```
Files to review: [FILES_CHANGED from discovery above]
Focus: correctness, edge cases, maintainability, Brighte standards,
test coverage adequacy, lessons.md verification.
Return: APPROVE / REQUEST CHANGES with line references.
Mark "Should Fix" (🟡) and "Consider" (🟢) items explicitly as /simplify candidates
— these are consumed by feature.md Step 8.5, not auto-remediated by the gate.
Cap tool calls at 12.
```

**1C. infra-reviewer (Sonnet) — conditional**
```
IF IAC_FILES is non-empty (from discovery above):
  → Files to review: [IAC_FILES]
  → Check: correct account, ap-southeast-2 region, VPC peering topology,
    IAM least-privilege, no public exposure of private resources.
  → Return: APPROVE / REQUEST CHANGES with account/region violations.
  → Cap tool calls at 10.
ELSE:
  → Skip. Record: "infra-reviewer: N/A"
```

Skip: code-security (deep scan) for Tier 2 unless security-reviewer returns HIGH or CRITICAL.
If security-reviewer finds HIGH or CRITICAL → escalate: run code-security as well before Wave 2.

---

### TIER 3 — Wave 1: 4 agents (all mandatory)

**1A. security-reviewer (Opus)**
```
Files to review: [FILES_CHANGED from discovery above]
Full checklist with heightened scrutiny:
- Auth0/Okta JWT: alg confusion, aud/iss validation, cross-env token acceptance
- MSSQL parameterisation (WAF is COUNT-only — app layer is the only defence)
- PHP Comms: injection, eval, variable includes
- EventBridge payload injection
- SSRF targeting internal GraphQL or Finpower endpoints
- Secrets, PII in logs, data residency
Return: findings + 0-100 score + WAF gap exposure.
Cap tool calls at 20.
```

**1B. code-security (Opus) — deep scan**
```
Files to review: [FILES_CHANGED from discovery above]
Full UNDERSTAND → HUNT → VERIFY → REPORT cycle.
Focus on: business logic flaws specific to Brighte's loan/payment domain,
broken access control (IDOR on loan/customer IDs), Auth0/Okta JWT exploitation,
MSSQL injection (WAF COUNT-only — critical), EventBridge consumer exploitation,
payment flow manipulation, data flow tracing end-to-end.
Return: full scored report 0-100 with confidence ratings per finding.
Cap tool calls at 25.
```

**1C. code-reviewer (Sonnet)**
```
Files to review: [FILES_CHANGED from discovery above]
Focus: correctness, edge cases, maintainability, Brighte standards,
test adequacy for high-risk paths, lessons.md verification.
Return: APPROVE / REQUEST CHANGES with line references.
Mark "Should Fix" (🟡) and "Consider" (🟢) items explicitly as /simplify candidates
— these are consumed by feature.md Step 8.5, not auto-remediated by the gate.
Cap tool calls at 15.
```

**1D. infra-reviewer (Sonnet) — conditional**
```
IF IAC_FILES is non-empty (from discovery above):
  → Files to review: [IAC_FILES]
  → Check: account ID, ap-southeast-2 region, VPC peering topology,
    WAF rules COUNT vs BLOCK status, IAM roles, Prod Finpower subnet placement,
    no direct Finpower ↔ Prod General peering assumed.
  → Return: APPROVE / REQUEST CHANGES.
  → Cap tool calls at 15.
ELSE:
  → Skip. Record: "infra-reviewer: N/A"
```

---

## WAVE 1 — Auto-remediation (all tiers)

After collecting all Wave 1 results, apply fixes autonomously for:
- All CRITICAL findings from any agent
- All HIGH findings from any agent
- All must-fix items from code-reviewer
- All critical/must-fix from infra-reviewer

After fixes:
- Re-run the affected tests to confirm nothing broke
- Note each fix in the gate report

**Hard stop:** If a CRITICAL finding cannot be auto-remediated (requires architectural decision),
STOP immediately. Surface to engineer with full context. Do not proceed to Wave 2.

---

## WAVE 2 — Run per tier

### TIER 1 — Wave 2: lessons-checker only

**2A. lessons-checker (inline — no subagent)**
```
Read tasks/lessons.md in full.
For every lesson entry, check the current implementation does not repeat the mistake.
List each lesson: CLEAR or REPEATED.
If REPEATED: treat as HIGH finding, fix before proceeding.
```

Skip: compliance-checker for Tier 1.

---

### TIER 2 — Wave 2: compliance (abbreviated) + lessons-checker

**2A. compliance-checker (Opus) — abbreviated**
```
Scope: changed files + Wave 1 gate report.
Focus: SOC 2 change management controls, data residency spot-check,
any unexpected APRA exposure surfaced during Wave 1.
Skip full APRA deep-dive unless Wave 1 found regulated data handling.
Return: PASS / CONDITIONAL PASS / FAIL with specific required actions.
Cap tool calls at 8.
```

**2B. lessons-checker (inline)**
```
Read tasks/lessons.md. Verify no past mistakes repeated.
List each lesson: CLEAR or REPEATED.
```

---

### TIER 3 — Wave 2: full compliance + lessons-checker

**2A. compliance-checker (Opus) — full**
```
Scope: changed files + Wave 1 gate report (including all remediations applied).
Full APRA CPS 234 + SOC 2 assessment:
- Data residency: ap-southeast-2 (AWS) / Australia East (Azure) confirmed?
- Auth0/Okta: customer PII data residency, breach notification risk (72hr window)
- Entra vs Auth0/Okta distinction respected in implementation?
- Audit logging present for all APRA-regulated operations?
- WAF gap exposure: which findings remain unmitigated by COUNT-only rules?
- SOC 2 CC6/CC7/CC8/A1/C1 controls maintained?
- Payment data handling: PCI-DSS adjacent controls respected?
Return: PASS / CONDITIONAL PASS / FAIL with required actions.
Cap tool calls at 12.
```

**2B. lessons-checker (inline)**
```
Read tasks/lessons.md. Verify no past mistakes repeated.
List each lesson: CLEAR or REPEATED.
If REPEATED: HIGH finding, fix before proceeding.
```

---

## Composite Score Calculation

### Tier 1
```
Security Score  = security-reviewer (0-100)
Quality Score   = code-reviewer → APPROVE=100, REQUEST CHANGES=70-(10 per must-fix)
Lessons Score   = ALL CLEAR=100, each REPEATED=-20

Composite = (Security × 0.50) + (Quality × 0.40) + (Lessons × 0.10)
```

### Tier 2
```
Security Score     = security-reviewer (0-100)
Quality Score      = code-reviewer verdict → APPROVE=100, REQUEST CHANGES=70-(10 per must-fix)
Infra Score        = infra-reviewer → APPROVE=100, N/A=100, REQUEST CHANGES=60
Compliance Score   = compliance-checker → PASS=100, CONDITIONAL=80, FAIL=0
Lessons Score      = ALL CLEAR=100, each REPEATED=-20

Composite = (Security × 0.40) + (Quality × 0.25) + (Infra × 0.15) + (Compliance × 0.15) + (Lessons × 0.05)
```

### Tier 3
```
Security Score     = security-reviewer (0-100)
Deep Scan Score    = code-security (0-100)
Quality Score      = code-reviewer → APPROVE=100, REQUEST CHANGES=70-(10 per must-fix)
Infra Score        = infra-reviewer → APPROVE=100, N/A=100, REQUEST CHANGES=60
Compliance Score   = compliance-checker → PASS=100, CONDITIONAL=80, FAIL=0
Lessons Score      = ALL CLEAR=100, each REPEATED=-20

Composite = (Security × 0.25) + (Deep Scan × 0.25) + (Quality × 0.20) + (Infra × 0.10) + (Compliance × 0.15) + (Lessons × 0.05)
```

### Score thresholds (all tiers)
- **≥ 85** → ✅ GATE PASS — proceed to PR
- **70–84** → ⚠️ CONDITIONAL PASS — surface MEDIUM findings, proceed after engineer acknowledgement
- **< 70** → 🔴 GATE FAIL — do not create PR, fix and re-run

---

## Gate Report Format

```
╔══════════════════════════════════════════════════════════════════╗
║  BRIGHTE QUALITY GATE REPORT                                     ║
║  Tier: [1 / 2 / 3] | Feature: [ticket + title]                  ║
║  Files reviewed: [N] | Est. tokens used: ~[N]k                  ║
╠══════════════════════════════════════════════════════════════════╣
║  WAVE 1                                                          ║
║  security-reviewer:  [score]/100  [PASS/REVIEW/FAIL]            ║
║  code-security:      [score]/100  [PASS/REVIEW/FAIL] | Tier 3   ║
║  code-reviewer:      [APPROVE / REQUEST CHANGES]                 ║
║  infra-reviewer:     [APPROVE / N/A / REQUEST CHANGES]          ║
║                                                                  ║
║  Auto-remediated:    [N findings fixed] | [list brief]          ║
║  /simplify candidates: [N yellow/green items — or "None"]       ║
║  WAF gap exposure:   [YES — list / NO]                          ║
╠══════════════════════════════════════════════════════════════════╣
║  WAVE 2                                                          ║
║  compliance-checker: [PASS / CONDITIONAL / FAIL / N/A Tier 1]  ║
║  lessons-checker:    [ALL CLEAR / N repeated — list]           ║
╠══════════════════════════════════════════════════════════════════╣
║  COMPOSITE: [score]/100                                          ║
║  GATE:      [✅ PASS / ⚠️ CONDITIONAL / 🔴 FAIL]                ║
╠══════════════════════════════════════════════════════════════════╣
║  ENGINEER ACTION REQUIRED                                        ║
║  [Items needing human decision — or "None"]                     ║
║                                                                  ║
║  PR CREATION: [APPROVED / BLOCKED — reason]                     ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## Hard Blocks — Gate ALWAYS fails regardless of composite score

- Any unresolved CRITICAL security finding
- compliance-checker returns FAIL (all tiers where it runs)
- Any lesson from lessons.md is REPEATED and unresolved
- Tests are failing
- Secrets detected in any committed file
- infra-reviewer finds resource in wrong region (non ap-southeast-2 / non Australia East)
- infra-reviewer finds VPC peering not in the approved topology
- WAF BLOCK rule changed to COUNT without security team sign-off documented in the gate report
- Tier 3 only: Auth0/Okta JWT validation missing on any protected endpoint
- Tier 3 only: Finpower/MSSQL query using string concatenation (not parameterised)
