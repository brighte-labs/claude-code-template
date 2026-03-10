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

## CI Automation — Claude reviews every PR automatically

The template includes `.github/workflows/claude-review.yml` — a GitHub Actions workflow that runs Claude agents against every PR push and posts a scored review comment.

### Setup (one-time per repo)

1. **Add `ANTHROPIC_API_KEY` as a secret** — set it once at the `brighte-labs` org level and all repos inherit it automatically, or add it to the individual repo under Settings → Secrets → Actions.

2. **Copy the workflow into your repo:**
```bash
cp claude-code-template/.github/workflows/claude-review.yml your-repo/.github/workflows/
```

3. Push — the workflow fires on the next PR open or push.

### How it works

1. **Tier detection** — inspects changed files to assign a risk tier:
   - Tier 3: `terraform/`, `.tf`, `iam`, `auth`, `payment` files
   - Tier 2: application code (anything not docs/tests/assets)
   - Tier 1: docs, tests, CSS, static assets only

2. **Claude review** — installs Claude Code CLI, runs agents against the PR diff:
   - All tiers: `code-reviewer`, `security-reviewer`
   - Tier 2 + 3: `infra-reviewer` (IaC files only)
   - Tier 3 only: `code-security`, `compliance-checker`

3. **PR comment** — posts a scored markdown report as a PR comment. On subsequent pushes, the comment is **updated in place** (no spam).

### Approximate cost per PR

| Tier | Agents | Cost |
|---|---|---|
| Tier 1 | code-reviewer + security-reviewer | ~$0.30–0.50 |
| Tier 2 | + infra-reviewer (if IaC) | ~$0.50–1.00 |
| Tier 3 | + code-security + compliance-checker | ~$3–5 |

> Drafts are skipped — the workflow only runs on ready PRs.

---

## Team

TechOps — [Brighte Labs](https://github.com/brighte-labs)
