---
description: Incident response pipeline. Diagnose → Root cause → Fix → Post-mortem. Use when something is broken in production or causing user impact.
allowed-tools: Task, Read, Write, Edit, Bash, Grep, Glob
---

# 🚨 Incident Response: $ARGUMENTS

This is incident mode. Speed and accuracy both matter. No guessing — find evidence.

## Step 1: Triage (immediate)

Answer these questions NOW before doing anything else:
- What is broken? (symptoms)
- Who is affected? (scope)
- When did it start? (timeline)
- What changed recently? (last deployments, config changes)
- What is the blast radius if we don't act?

**Severity:**
- P1 = Payment processing, complete outage, data breach risk → escalate NOW
- P2 = Major feature broken, many users affected → fix within 1 hour
- P3 = Partial degradation, workaround exists → fix within 4 hours

## Step 2: Parallel Diagnosis

Launch these subagents IN PARALLEL:

1. **Researcher subagent**: Search codebase and logs for evidence related to: $ARGUMENTS
2. **Researcher subagent** (second instance): Check recent git commits, config changes, and deploy history

Synthesise findings into a hypothesis: "Most likely cause is X because of evidence Y and Z"

## Step 3: Root Cause

Do NOT apply fixes until you have confirmed the root cause.
Evidence required before fixing:
- [ ] Log entry or error trace showing the failure
- [ ] Confirmed timeline of when it started
- [ ] Hypothesis tested (not just assumed)

## Step 4: Fix

Implement the minimal fix that resolves the root cause.
- Prefer reversible changes
- If uncertain: implement a safe rollback first, then the fix
- Test the fix against the symptom before declaring resolved

Use the **security-reviewer subagent** if the fix touches auth, payments, or PII.

## Step 4a: Review the Fix

Use the **code-reviewer subagent** to review the fix before proceeding. Hotfixes written under pressure are the highest-risk code in any codebase — a fresh pair of eyes is non-negotiable.

Apply any CRITICAL or HIGH findings before moving to verification.

## Step 5: Verify Resolution

Confirm the incident is resolved:
- [ ] Original symptom no longer present
- [ ] No new errors introduced
- [ ] Monitoring shows return to normal
- [ ] Affected users can complete their workflows

## Step 6: Post-Mortem (always)

Write to `tasks/done.md`:

```markdown
## Incident: $ARGUMENTS
**Date:** [today]
**Duration:** [start → resolution]
**Severity:** P[1/2/3]
**Root cause:** [one sentence]
**Fix applied:** [what was changed]
**Timeline:** [key events]
**What we learned:** [insights]
**Prevention:** [what would have caught this earlier]
**Action items:** [ ] [specific follow-up tasks]
```

Also add to `tasks/lessons.md` if there's a pattern to avoid repeating.
