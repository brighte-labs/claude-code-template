# /start — Brighte Session Initialisation
# Usage: /start
# Run this at the beginning of every Claude Code session.
# Sets context, declares behaviour, and orients Claude for the session ahead.

---

## Step 1 — Load All Context

Read the following files in order before doing anything else:

```
CLAUDE.md           → full Brighte operating context (always reload — never assume it's cached)
tasks/todo.md       → pick up any in-progress work
tasks/lessons.md    → load all past corrections — apply immediately for this session
tasks/memory.md     → key file paths, architectural decisions, project gotchas
```

If any of these files don't exist, create them as empty files now.

---

## Step 2 — Declare Session Behaviour

Print the following session declaration to the engineer. This is not optional — it sets
expectations clearly so engineers know exactly how Claude will behave this session:

```
╔══════════════════════════════════════════════════════════════════╗
║  CLAUDE CODE — BRIGHTE SESSION STARTED                          ║
╠══════════════════════════════════════════════════════════════════╣
║  Context loaded:                                                 ║
║  ✅ CLAUDE.md — Brighte infrastructure + compliance context     ║
║  ✅ lessons.md — [N] past corrections loaded                    ║
║  ✅ memory.md — project state loaded                            ║
║  ✅ todo.md — [in-progress tasks / clean]                       ║
╠══════════════════════════════════════════════════════════════════╣
║  HOW I HANDLE REQUESTS THIS SESSION                             ║
║                                                                  ║
║  Slash commands → structured pipeline                           ║
║    /feature [ticket] "[description]"  → full feature pipeline  ║
║    /review-pr [PR number]             → full quality gate       ║
║    /security-scan [path]              → deep security audit     ║
║    /incident "[description]"          → incident response       ║
║    /simplify [path]                   → refactor & clean up     ║
║    /batch "[rule]" [scope]            → parallel bulk transform ║
║    /summarise-pr [PR number]          → walkthrough + diagram   ║
║    /setup [repo-path]                 → onboard a new repo      ║
║    /start                             → this session init       ║
║                                                                  ║
║  Natural language code changes → auto-routed to /feature       ║
║    "improve the Dockerfile"           → treated as /feature     ║
║    "fix the bug in payments.ts"       → treated as /feature     ║
║    "add port 5000 to the app"         → treated as /feature     ║
║    "update the requirements.txt"      → treated as /feature     ║
║    "migrate all files to..."          → treated as /batch       ║
║    "clean up / simplify this code"    → treated as /simplify    ║
║                                                                  ║
║  Read-only requests → answered conversationally                 ║
║    "explain this function"            → direct answer           ║
║    "how should I approach X"          → advice only             ║
║                                                                  ║
║  EVERY code change goes through the mandatory quality gate.     ║
║  No PR is created without: security scan, compliance check,     ║
║  code review, lessons check, and composite score ≥ 70.         ║
╠══════════════════════════════════════════════════════════════════╣
║  HARD BLOCKS ACTIVE                                              ║
║  🔴 git push origin main             → always blocked           ║
║  🔴 git push --force                 → always blocked           ║
║  🔴 hardcoded secrets                → always blocked           ║
║  🔴 non ap-southeast-2 deployments   → always blocked           ║
║  🔴 direct Finpower ↔ Prod General   → flagged (no peering)    ║
╠══════════════════════════════════════════════════════════════════╣
║  READY. What are we building today?                             ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## Step 3 — Surface Any In-Progress Work

If `tasks/todo.md` has incomplete items:

```
⚠️  In-progress work detected:
[list incomplete todo items with ticket IDs]

Continue previous work, or start something new?
```

Wait for engineer response before proceeding.

If `tasks/todo.md` is clean: proceed directly — no prompt needed.

---

## Step 4 — Surface Any Pending Lessons

If `tasks/lessons.md` was updated in the last session, highlight the most recent 3 entries:

```
📖 Recent lessons (applied this session):
- [date] [category]: [lesson summary]
- [date] [category]: [lesson summary]
- [date] [category]: [lesson summary]
```

This keeps past mistakes front of mind without the engineer having to ask.
