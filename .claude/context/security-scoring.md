# Security Scoring Reference
# Used by: code-security, security-reviewer
# Consistent scoring across all security agents.

## Score Calculation

Start at 100. Deduct based on validated findings:
- CRITICAL: -25 each
- HIGH: -15 each
- MEDIUM: -8 each
- LOW: -3 each
- INFORMATIONAL: -0

Floor at 0. Cap at 100.

**WAF gap modifier:** If any findings exist in Finpower/MSSQL code paths where WAF is COUNT-only, note that the effective risk is higher than the score suggests — application-layer controls are the only prevention.

## Score Bands

- 90–100: ✅ PASS — good security posture
- 75–89:  ⚠️ REVIEW — address High findings before merge
- 50–74:  🔴 FAIL — Critical/High findings must be fixed
- 0–49:   🚨 CRITICAL FAIL — do not merge

## Score Dashboard Format

```
╔══════════════════════════════════════════════════╗
║  BRIGHTE SECURITY SCORE: [X]/100                 ║
║  Rating: [CRITICAL/HIGH/MEDIUM/LOW/PASS]         ║
║  Findings: [C] Critical [H] High [M] Med [L] Low ║
║  WAF Gap Exposure: [YES / NO]                    ║
╚══════════════════════════════════════════════════╝
```
