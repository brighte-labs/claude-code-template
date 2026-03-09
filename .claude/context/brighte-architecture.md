# Brighte Architecture & Security Context
# Single source of truth — referenced by agents that need infrastructure awareness.
# Update this file when infrastructure changes. All agents inherit updates automatically.

---

## Dual Identity System — Critical Distinction

Brighte operates **two separate identity systems** with different APRA obligations:

**Microsoft Entra ID** — Employee identity
- Scope: staff, contractors, internal tools, M365
- Data residency: Australia East (Azure) — compliant
- Breach obligation: standard NOTIFIABLE DATA BREACH (NDB) scheme

**Auth0/Okta** — Customer/vendor/partner application identity
- Scope: all external-facing portals (consumer app, vendor portal, installer app, enterprise portal, marketplace)
- ⚠️ **APRA obligation:** Customer PII (names, emails, financial identifiers) lives in Auth0/Okta — data residency must be confirmed as Australian
- ⚠️ **Breach obligation:** Any Auth0/Okta breach exposes customer financial data — APRA notification obligations apply within 72 hours

**Rule:** When reviewing auth-related changes, always determine: is this the employee identity system (Entra) or the customer identity system (Auth0/Okta)? Confusing these is a critical finding.

---

## AWS Account Structure

All production workloads are in **ap-southeast-2 (Sydney)**.

| Account | ID | CIDR | What lives here |
|---|---|---|---|
| Control Mgmt | 040925256309 | 10.37.0.0/16 | Zscaler connectors (x4), OpenVPN (x2), Stitch bastion, Finpower gateway, Transit subnets |
| Labs | 281902667290 | 10.47.0.0/16 | Experimentation — lower security bar acceptable |
| Staging | 386594967457 | 10.57.0.0/16 | Pre-prod — must mirror prod security controls |
| UAT | 128555121632 | 10.67.0.0/16 | UAT — must mirror prod security controls |
| Prod General | 590257951365 | 10.77.0.0/16 | Primary production — 34 ECS services, 139 Lambda functions, 15 RDS, 60 S3 buckets, 21 CloudFront distributions |
| Staging Analytics | 054429138773 | 10.87.0.0/16 | Analytics staging |
| Prod Analytics | 610755455255 | 10.97.0.0/16 | Apache Airflow (analytics-mimir-prod), 23 S3 buckets |
| UAT Finpower | 450567618273 | 10.102.0.0/16 | Finpower UAT |
| Prod Finpower | 114211845990 | 10.103.0.0/16 | Finpower LMS — MSSQL 2017, Windows DCs, Entra Connect |

> ⚠️ **Admin AWS account (TechOps-managed):** Production Route 53 hosted zones and CloudFront distributions live in a separate Admin AWS account managed by TechOps. Do not create or modify prod Route 53 or CloudFront resources in application IaC.

Labs (281902667290) must NOT contain production customer data.

---

## VPC Subnet Pattern (every account)

- **Public:** x.x.0.0/20, x.x.16.0/20, x.x.32.0/20 — internet-facing only
- **Private:** x.x.80.0/20, x.x.96.0/20, x.x.112.0/20 — app tier
- **Secure:** x.x.160.0/20, x.x.176.0/20, x.x.192.0/20 — databases, secrets
- **Transit (Mgmt only):** 10.37.120.0/21, 10.37.128.0/21, 10.37.136.0/21

---

## VPC Peering Topology

- ✅ Prod General ↔ Control Mgmt
- ✅ Prod Analytics ↔ Control Mgmt
- ✅ Prod Finpower ↔ Control Mgmt
- ✅ Prod General ↔ Prod Analytics
- ❌ **NO direct Prod Finpower ↔ Prod General** — all cross-account traffic routes via Mgmt
- ❌ **NO direct account-to-account peering** outside the above — new peering requires explicit approval

---

## WAF Configuration — Known Security Debt

AWS WAF is on both CloudFront and ALB across all accounts. **Critical issue: most rules are in COUNT mode (monitoring only), not BLOCK.**

**Rules in COUNT (monitoring only) — not enforcement:**
- `AWSManagedRulesAmazonIpReputationList` (Finpower accounts)
- `AWSManagedRulesSQLiRuleSet` (Finpower accounts)
- `AWSManagedRulesWindowsRuleSet` (Finpower accounts)
- `brighte-rules-sql` (prod-general ALB)
- `brighte-rules-common` (staging ALB — count override)
- `AWSManagedRulesBotControlRuleSet` (Finpower — count)

**Rules that are actively BLOCKING:**
- `Block-nonzscaler-ip-v2graphql` — blocks non-Zscaler IPs from `/v2/internal/graphql` (prod, staging, UAT)
- `Whitelisted-IPs` — ALLOW action on CloudFront for trusted IPs
- `brighte-rules-finpower-exclude` / `brighte-rules-flatrate-exclude` — ALLOW specific IP/path combos

**WAF change rules:**
1. Document the business justification for COUNT vs BLOCK
2. Flag COUNT rules as security debt requiring remediation
3. Never change BLOCK rules to COUNT without security sign-off

---

## Application Architecture

**Public entry points:**
- **Brighte Gateway** — central API gateway for all UX portals
- **CloudFront (21 distributions)** → ALB → ECS services in Prod General
- **Auth0/Okta** — handles all external authentication before hitting the gateway

**Core internal services (Prod General — ECS/Lambda):**
- **Finance Core** — lending logic, core financial processing
- **User Service** — user management
- **Application Service** — loan applications
- **Scheduler Service** — scheduled jobs
- **Installer Service** — installer/vendor management
- **Enterprise Service** — enterprise partner management

**Event-driven layer:**
- **EventBridge** — internal event bus connecting all core services to bridge services
- **Bridge services** (consume EventBridge): CRM (→Salesforce), CDP (→Segment), Comms Lambda (→SendGrid), PHP Comms (→Twilio), LMS (→Finpower), UV Checker (→ACT gov), Payments (→Pylon/Fatzebra)

**Analytics (Prod Analytics account):**
- **Apache Airflow** (analytics-mimir-prod) — data pipeline orchestration

**Finpower (Prod Finpower account — legacy Windows stack):**
- **LMS** — loan management system, accessed via EventBridge from Prod General
- **MSSQL 2017** (primary + read replica RDS)
- **Windows servers:** PROD-AZURECONNECT01 (Entra Connect), PROD-WKSDC01 (Workspaces DC), PROD-FP-WEB01-2022, PROD-FP-WCC-2022 (Cloud Connect), PROD-FP-WEB2RO-2022
- ⚠️ PHP Comms service is flagged as technical debt — treat PHP code with extra security scrutiny

**Known sensitive paths:**
- `https://api.brighte.com.au/v2/internal/graphql` — Zscaler-IP-restricted, never called from customer-facing code
- `https://portal.brighte.com.au/rest/v1/vendor/session` — 2-IP whitelist, POST only
- Finance Core — all paths touch regulated financial data
- Payments bridge → Fatzebra/Pylon — PCI DSS adjacent

**Third-party integrations:**
- Salesforce (CRM), Segment (CDP), SendGrid (email), Twilio (SMS via PHP Comms)
- Finpower/ACT gov/Fortiro (compliance/verification), Fatzebra/Pylon (payments)
- Zendesk, Google Maps, OpenSolar, ABN lookup

---

## Azure Services

- **Microsoft 365:** Teams, SharePoint, Exchange, Entra ID
- **Azure OpenAI:** AI services (Maester assistant, other AI tooling)
- **Logic Apps:** Automation workflows
- **Australia East region** for all regulated Azure workloads

---

## CI/CD & Tooling

- **Code:** GitHub (all repos)
- **CI/CD:** GitHub Actions — all secrets via GitHub Secrets or OIDC
- **ITSM:** Freshservice (primary IT), Jira (engineering projects)
- **Docs:** Confluence + SharePoint
- **Comms:** Microsoft Teams (primary), Slack (some teams)

---

## Compliance Summary

- **APRA CPS 234:** Data residency in Australia, encryption at rest/transit, audit logging
- **SOC 2 Type 2:** Change management, access controls, availability, confidentiality
- **PCI DSS adjacent:** Payment data handling — extra scrutiny required
- **Auth0/Okta:** Separate APRA assessment needed — customer PII lives here

---

## Prod Finpower — Special Scrutiny

- Windows Domain Controllers should be in **Private or Secure subnets only**
- MSSQL 2017 RDS must be in **Secure subnet** with encryption at rest
- Any public subnet resources in Finpower need documented justification
- No direct peering to Prod General — flag any IaC that implies it does
- WAF is COUNT-only — application-layer defences are the only real prevention for SQL injection
