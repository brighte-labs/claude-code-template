---
description: Run a full AI-powered security scan on code, a feature, PR, or entire codebase. Reasons about vulnerabilities like a human security researcher — catches business logic flaws, broken access control, and complex data flow issues that static tools miss. Returns a scored report with severity ratings, confidence levels, and suggested patches. Nothing is applied without your approval.
allowed-tools: Task, Read, Grep, Glob, Bash
---

# Security Scan: $ARGUMENTS

Use the **code-security subagent** to run a full security analysis on: $ARGUMENTS

The subagent will:
1. Build a mental model of the code's architecture and data flows
2. Hunt for vulnerabilities through reasoning, not just pattern matching
3. Verify every finding with a complete exploit path before reporting
4. Score the codebase and produce a prioritised findings report

If no specific scope is provided, scan all non-test application code in the current repository.

When complete, present the full security dashboard with score, findings, and next steps.
Remind the user that **no fixes will be applied without their explicit approval**.
