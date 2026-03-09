# claude-code-template

Brighte's Claude Code agentic engineering stack — copy this into any repo to get the full pipeline.

## What's included

```
.claude/
  agents/       14 specialist subagents (security, compliance, infra, review, test, etc.)
  commands/     11 slash command pipelines (/feature, /quality-scan, /review-pr, etc.)
  context/      Brighte architecture + security scoring reference
  settings.json Safety permissions + hooks (blocks git push --force, rm -rf, etc.)
CLAUDE.md       The AI operating system — loaded every session automatically
```

## How to adopt

### 1. Copy into your repo

```bash
git clone https://github.com/brighte-labs/claude-code-template.git
cp -r claude-code-template/.claude your-repo/
cp claude-code-template/CLAUDE.md your-repo/
```

### 2. Customise CLAUDE.md for your service

At the bottom of `CLAUDE.md` there is a `## 📦 This Service` section. Replace it with your service's context:

- Service name, port, staging URL
- Architecture (`src/` layout, entry point, route pattern)
- How to add a new endpoint
- How to run locally / run tests
- Key files table
- Coding conventions specific to your service
- SSM secret convention
- What NOT to do

The rest of CLAUDE.md (Core Directive, Brighte Loop, Subagents, Security, Infrastructure, Code Standards, Hard Stops, Routing) is generic — leave it as-is.

### 3. Create your tasks/ folder

```bash
mkdir tasks
touch tasks/todo.md tasks/lessons.md tasks/memory.md tasks/done.md
```

Claude will populate these as you work.

### 4. Start your first session

```
/start
```

---

## Slash commands

| Command | What it does |
|---|---|
| `/start` | Session init — loads context, surfaces in-progress work |
| `/feature [TICKET] "description"` | Full feature pipeline: plan → TDD → implement → gate → PR |
| `/quality-scan` | Full repo audit — static analysis, test coverage, CVEs, duplication |
| `/review-pr [PR number]` | Full agent-backed PR review |
| `/security-scan [path]` | Standalone deep security scan |
| `/simplify [path]` | Code cleanup — no new features |
| `/batch "[rule]" [scope]` | Bulk transformation across many files |
| `/summarise-pr [PR]` | PR walkthrough + Mermaid architecture diagram |
| `/incident "[description]"` | Incident response: diagnose → root cause → fix → post-mortem |
| `/setup [repo-path]` | Onboard a new repo |

## Agents (invoked by Claude, not by you)

| Agent | Model | When it fires |
|---|---|---|
| `researcher` | Haiku | Research phase — every `/feature` |
| `security-reviewer` | Opus | Quality gate — every tier |
| `code-security` | Opus | Quality gate — Tier 3 only (IaC/payments/PII) |
| `code-reviewer` | Sonnet | Quality gate — every tier |
| `infra-reviewer` | Sonnet | Quality gate — Tier 2/3 (when IaC touched) |
| `compliance-checker` | Opus | Quality gate — Tier 2/3 |
| `test-writer` | Sonnet | When new business logic is added |
| `pr-summary` | Sonnet | After gate passes — walkthrough + diagram |
| `static-analysis` | Sonnet | `/quality-scan` only |
| `test-coverage` | Sonnet | `/quality-scan` only |
| `dependency-audit` | Sonnet | `/quality-scan` only |
| `code-duplication` | Sonnet | `/quality-scan` only |
| `code-transformer` | Sonnet | `/batch` only |
| `feature-developer` | Sonnet | Full delegation mode |

## Risk tier system

| Tier | Triggers | Gate time | Tokens |
|---|---|---|---|
| Tier 1 | Docs, tests, CSS, static assets | ~4 min | ~35k |
| Tier 2 | App code, new endpoints, dependencies | ~8 min | ~70k |
| Tier 3 | Terraform, Auth0/Okta, payments, PII, IAM | ~17 min | ~250k |

## Team

TechOps — [Brighte Labs](https://github.com/brighte-labs)
