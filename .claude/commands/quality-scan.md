---
description: Full code quality scan covering static analysis, test coverage, code duplication, and dependency vulnerabilities. Auto-detects language and tooling (Python, JavaScript/TypeScript, PowerShell, Go, PHP, Bash, Terraform/HCL, Dockerfile, GitHub Actions) then synthesises results into a scored quality dashboard. Nothing is fixed without your approval.
allowed-tools: Bash, Read, Grep, Glob, Task
---

# Quality Scan: $ARGUMENTS

Run a full quality audit on the scope provided. If no scope is given, scan the entire repository.

**Nothing is fixed without explicit user approval.**

---

## Step 0: Confirm Before Running

Before proceeding, present the following tradeoff and ask for confirmation:

> **Quality Scan mode:**
>
> | Mode | Speed | Output |
> |---|---|---|
> | `/quality-scan` (this command) | Slower — runs 4 subagents then synthesises | Unified dashboard, composite score, single Must Fix list |
> | Run subagents individually | Faster — agents run on demand | Raw reports per agent, you read 4 separate outputs |
>
> Running `/quality-scan` now. This will take longer than running subagents individually but gives you a single scored dashboard and one consolidated Must Fix list.
>
> **Proceed? (yes / no)**

Wait for the user to confirm before continuing. If they say **no** or want the faster option, list the individual subagents they can invoke:
- `static-analysis` — linting and security linting
- `test-coverage` — test suite and coverage %
- `dependency-audit` — CVE scan across package managers
- `code-duplication` — duplication clusters and refactor suggestions

---

## Step 1: Detect Languages & Tooling

Fingerprint the repository scoped to `$ARGUMENTS` (or `.` if no scope given).
Use `$SCOPE` as the base path for all discovery below.

```bash
SCOPE="${ARGUMENTS:-.}"

# Python
find "$SCOPE" -name "*.py" \
  -not -path "*/venv/*" -not -path "*/.venv/*" -not -path "*/__pycache__/*" \
  | head -5
test -f pytest.ini || test -f pyproject.toml || test -f setup.cfg && echo "pytest config found"

# JavaScript / TypeScript
find "$SCOPE" -name "package.json" -not -path "*/node_modules/*" | head -3
find "$SCOPE" \( -name "jest.config.*" -o -name "vitest.config.*" -o -name "tsconfig.json" \) | head -3

# PowerShell
find "$SCOPE" \( -name "*.ps1" -o -name "*.psm1" \) | head -5
find "$SCOPE" \( -name "*.Tests.ps1" -o -name "*.test.ps1" \) | head -3

# Go
find "$SCOPE" -name "go.mod" | head -3

# PHP
find "$SCOPE" \( -name "composer.json" -o -name "phpunit.xml" \) | head -3
find "$SCOPE" -name "*.php" -not -path "*/vendor/*" | head -5

# Bash / Shell
find "$SCOPE" -name "*.sh" | head -5

# Terraform / HCL
find "$SCOPE" -name "*.tf" -not -path "*/.terraform/*" | head -5
test -f .tflint.hcl && echo "tflint config found"

# Dockerfile
find "$SCOPE" -name "Dockerfile*" -o -name "*.dockerfile" | head -5

# GitHub Actions
find "$SCOPE" -path "*/.github/workflows/*.yml" -o -path "*/.github/workflows/*.yaml" | head -5

# Catch-all: find any file extensions NOT already covered above
# Excludes: known extensions, binary files, generated dirs, lock files
find "$SCOPE" -type f \
  -not -path "*/node_modules/*" -not -path "*/.terraform/*" \
  -not -path "*/venv/*" -not -path "*/.venv/*" -not -path "*/__pycache__/*" \
  -not -path "*/.git/*" -not -path "*/vendor/*" -not -path "*/dist/*" \
  -not -path "*/build/*" -not -path "*/coverage/*" \
  | grep -v -E "\.(py|js|jsx|ts|tsx|ps1|psm1|go|php|sh|tf|tfvars|md|json|yml|yaml|lock|sum|mod|toml|cfg|ini|txt|svg|png|jpg|gif|ico|woff|woff2|ttf|eot|pdf|zip|tar|gz|exe|dll|so|dylib|class|jar)$" \
  | grep -v "Dockerfile" \
  | sed 's/.*\.//' | sort -u
```

Build a **Language Map** from the results:

```
Language Map:
  Python:           YES / NO  (test runner: pytest / none found)
  JavaScript/TS:    YES / NO  (test runner: jest / vitest / none found)
  PowerShell:       YES / NO  (test runner: pester / none found)
  Go:               YES / NO  (test runner: go test / none found)
  PHP:              YES / NO  (test runner: phpunit / none found)
  Bash/Shell:       YES / NO  (no coverage tooling — shellcheck only)
  Terraform/HCL:    YES / NO  (tflint / terraform validate)
  Dockerfile:       YES / NO  (hadolint)
  GitHub Actions:   YES / NO  (actionlint)
  Unrecognised:     [list of extensions found, e.g. .rb .java .cs .rs] / NONE
```

**If Unrecognised extensions are found:**
- List each extension and a sample file path
- These files will receive AI-based review in Step 2 (no tool output available)
- Coverage for unrecognised languages = N/A (excluded from scoring)
- Flag in the dashboard: "X files in unrecognised languages — AI review only, no tooling"

---

## Step 2: Parallel Agent Analysis

Launch all four subagents **IN PARALLEL** using the Task tool.
Pass the scope and Language Map from Step 1 to each agent.

---

### 1. static-analysis subagent

Task: "Use the static-analysis subagent.

Scope: $ARGUMENTS (or the full repo if no scope was given).

Language Map:
[paste Language Map from Step 1]

Run static analysis linters for all detected languages. Fall back to AI-based review for any unrecognised extensions or missing tools. Return your structured report."

---

### 2. test-coverage subagent

Task: "Use the test-coverage subagent.

Scope: $ARGUMENTS (or the full repo if no scope was given).

Language Map:
[paste Language Map from Step 1]

Run test suites for all detected languages and report coverage %, files below 80%, and critical untested paths. Return your structured report."

---

### 3. dependency-audit subagent

Task: "Use the dependency-audit subagent.

Scope: $ARGUMENTS (or the full repo if no scope was given).

Language Map:
[paste Language Map from Step 1]

Audit dependencies for CRITICAL and HIGH CVEs across all detected package managers. Fall back to AI CVE knowledge if audit tools are unavailable. Return your structured report."

---

### 4. code-duplication subagent

Task: "Use the code-duplication subagent.

Scope: $ARGUMENTS (or the full repo if no scope was given).

Language Map:
[paste Language Map from Step 1]

Detect code duplication using available tools and AI analysis. Rate overall duplication (LOW/MEDIUM/HIGH) and list numbered clusters with refactor suggestions. Return your structured report."

---

## Step 3: Synthesise — Quality Dashboard

Combine results from all four subagents. Calculate scores using the Scoring Reference below.

**Coverage score calculation:**
- Use a weighted average by source file count across languages with tooling available
- Exclude Bash, Terraform, Dockerfile, and GitHub Actions from coverage scoring (no tooling)
- Do NOT use "lowest language" — this unfairly penalises polyglot repos

Present the complete audit results using the following markdown format. Replace all `[placeholders]` with actual data — omit any row or section where the language was not detected.

---

## 🔍 Brighte Code Quality Report

**Scope:** [files/directories scanned]
**Languages:** [detected from Language Map]
**Scanned:** [timestamp]

---

### Scorecard

| Dimension | Score | Status |
|---|---|---|
| Static Analysis | [X]/100 | ✅ PASS / ⚠️ REVIEW / 🔴 FAIL |
| Test Coverage | [X]% | ✅ PASS / ⚠️ REVIEW / 🔴 FAIL |
| Code Duplication | LOW / MED / HIGH | ✅ PASS / ⚠️ REVIEW / 🔴 FAIL |
| Dependency CVEs | NONE / N CVEs | ✅ PASS / 🔴 FAIL |
| **Overall** | **[X]/100** | **✅ PASS / ⚠️ REVIEW / 🔴 FAIL / 🚨 CRITICAL FAIL** |

> ℹ️ Security linting (bandit, checkov, gosec, eslint-security, etc.) is included in Static Analysis above.
> Full Brighte-contextual security review (Auth0/Okta, WAF gaps, IDOR, race conditions) runs in `/quality-gate` before merge.

---

### Static Analysis

[Only include rows for detected languages:]

| Language | Tool | Result |
|---|---|---|
| Python | flake8 | [N errors] |
| Python | pylint | [X.X/10] |
| Python | mypy | [N errors] |
| JS/TS | eslint | [N errors] |
| JS/TS | tsc | [N errors] |
| PowerShell | PSScriptAnalyzer | [N errors, N warnings] |
| Go | go vet | [N issues] |
| Go | staticcheck | [N issues] |
| PHP | phpstan | [N errors] |
| PHP | phpcs | [N violations] |
| Bash | shellcheck | [N errors, N warnings] |
| Terraform | tflint | [N issues] |
| Terraform | terraform validate | PASS / FAIL |
| Dockerfile | hadolint | [N errors, N warnings] |
| GitHub Actions | actionlint | [N errors] |

---

### Security Linting *(from static-analysis subagent)*

[Only include rows for detected languages:]

| Language | Tool | HIGH | MEDIUM |
|---|---|---|---|
| Python | bandit | N | N |
| JS/TS | eslint-security | N | N |
| JS/TS | npm audit | N CRITICAL | N HIGH |
| Go | gosec | N | N |
| PowerShell | PSScriptAnalyzer | N | N |
| PHP | — | N | N |
| Bash | shellcheck (injection) | N | N |
| Terraform | checkov | N CRITICAL | N HIGH |
| Dockerfile | hadolint | N | N |
| GitHub Actions | — | N | N |

> ℹ️ Tool-based security linting only. For Brighte-contextual review (WAF gaps, Auth0/Okta, IDOR, race conditions) run `/quality-gate` or `/security-scan`.

[If Finpower-path findings exist, add:]
> ⚠️ **Finpower-path findings:** [list — WAF is COUNT-only and cannot mitigate these]
> → Run `/security-scan` for full exploit-path analysis

[If Unrecognised languages were found, add this section — otherwise omit:]

#### Unrecognised Languages (AI review — no tooling)

| Extension | Files Reviewed | Static | Security |
|---|---|---|---|
| .[ext] | N | [N Must Fix, N Should Fix] | [N HIGH, N MEDIUM] |

> ℹ️ No static analysis tools available for these types — findings are AI-assessed, confidence MEDIUM.
> To get proper coverage, add these languages to your CI pipeline.

---

### Test Coverage

[Only include rows for detected languages — mark N/A for non-testable types:]

| Language | Runner | Coverage |
|---|---|---|
| Python | pytest | [X]% |
| JS/TS | jest / vitest | [X]% |
| PowerShell | Pester | [X]% |
| Go | go test | [X]% |
| PHP | PHPUnit | [X]% |
| Terraform | — | N/A |
| Dockerfile | — | N/A |
| Bash | — | N/A |
| GitHub Actions | — | N/A |

**Weighted average:** [X]% *(by source file count; N/A languages excluded)*

**Files below 80%:**
- `[file]` — [X]% ([N uncovered lines / functions])

**Critical gaps** *(untested auth / payment / DB / external API paths):*
- [list or "None identified"]

---

### Dependency Vulnerabilities

[Only include rows for detected package managers — omit section if no vulnerabilities:]

| Package | Version | Severity | CVE |
|---|---|---|---|
| [package] | [version] | 🔴 CRITICAL | CVE-XXXX-XXXXX |
| [package] | [version] | 🟠 HIGH | CVE-XXXX-XXXXX |

[If no vulnerabilities:] ✅ No CRITICAL or HIGH CVEs found.

---

### Code Duplication

**Clusters found:** [N] — **Rating:** LOW / MEDIUM / HIGH

[For each cluster:]
- `[file A]` ↔ `[file B]` — [N lines], [language] — *suggested: [refactor action]*

**Cross-language duplicates:** [any logic duplicated across languages, or "None"]

---

### 🔴 Must Fix — Blocks Merge

[Critical findings from all subagents: security HIGH+, CRITICAL/HIGH CVEs, 0% on critical paths, Terraform wrong region/account, exposed secrets in GHA, Finpower injection risks]

[Number and label each item. Use code spans for file references. Note source subagent:]

1. `[file:line]` — **[short title]** — [one-sentence description] *(source: [subagent])*
2. ...

[If none:] ✅ No blocking issues found.

---

### ⚠️ Should Fix — Non-Blocking

[Medium priority findings from all subagents + coverage gaps:]

1. `[file:line]` — **[short title]** — [one-sentence description] *(source: [subagent])*
2. ...

[If none:] ✅ No non-blocking issues found.

---

### Next Steps

1. **Approve a fix** — reply with `"Fix [item number or title]"`
2. **Write missing tests** — `"Write tests for [file]"` *(uses test-writer subagent)*
3. **Fix dependency CVEs** — `/dependency-audit` or approve individual upgrades
4. **Deep security analysis** — `/security-scan`
5. **Re-run after fixes** — `/quality-scan $ARGUMENTS`

> ⚠️ **All fixes require your explicit approval before application.**

---

## Scoring Reference

**Static Analysis score** (start 100, deductions per detected language only):
- Python: flake8 F-series −15 each; flake8 E-series −3 each (cap −20); pylint <7 −10, <5 −25; mypy logic errors −8 each (cap −20)
- JS/TS: eslint errors −5 each (cap −25); tsc errors −8 each (cap −20)
- PowerShell: PSScriptAnalyzer Error −15 each; Warning −3 each (cap −15)
- Go: go vet issues −10 each; staticcheck −5 each (cap −20)
- PHP: phpstan errors −8 each (cap −25); phpcs violations −2 each (cap −10)
- Bash: shellcheck SC1xxx −15 each; SC2xxx −5 each (cap −20)
- Terraform/HCL: tflint errors −10 each; terraform validate FAIL −25 (cap −25)
- Dockerfile: hadolint errors −8 each; warnings −2 each (cap −15)
- GitHub Actions: actionlint errors −10 each (cap −20)
- Security tool HIGH findings (bandit, gosec, checkov, eslint-security — from static-analysis): −15 each; MEDIUM: −5 each

**Coverage score**: weighted average by source file count across languages with tooling.
Languages with no coverage tooling (Bash, Terraform, Dockerfile, GitHub Actions) are excluded from this score entirely.

**Dependency vulnerabilities**: separate dimension — does not roll into Overall score but shown prominently.
- CRITICAL CVE present: overall score capped at 60
- HIGH CVE present: −10 per CVE (cap −20)

**Duplication** (from code-duplication subagent): LOW < 5% = no deduction | MEDIUM 5–15% = −10 | HIGH > 15% = −20

**Overall score**: weighted average — Static 40%, Coverage 40%, Duplication 20%
(Dependency CVE cap applied after weighted average)

Score bands:
- 90–100: ✅ PASS
- 75–89:  ⚠️ REVIEW — address before merge
- 50–74:  🔴 FAIL — must fix before merge
- 0–49:   🚨 CRITICAL FAIL — do not merge
