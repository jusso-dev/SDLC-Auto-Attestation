---
name: ssdf-attest
description: Generate a Secure Software Development attestation report assessing cyber posture of a code repository. Maps evidence to NIST SSDF v1.1 (SP 800-218), Australian ISM + Essential Eight, and OWASP SAMM/ASVS. Emits Markdown report + machine-readable JSON. Flags gaps and hands them to ssdf-remediate. Use when user asks for "attestation report", "security posture", "SSDF compliance", "cyber attestation", "is this repo secure".
argument-hint: "[repo path or --evidence <sdlc-docs/manifest.json>]"
---

# Secure Software Development Attestation

Produces an attestation report on a repository's cyber posture against three frameworks simultaneously:

1. **NIST SSDF v1.1 (SP 800-218)** — practices PO, PS, PW, RV
2. **Australian ISM controls + Essential Eight** maturity (ML0–ML3)
3. **OWASP SAMM v2** business functions + **ASVS v4 L2** verification requirements

## When to invoke

- User runs `/ssdf-attest [path]`
- User says: "attest this repo", "run attestation", "check secure dev posture", "compliance check", "audit security"
- After `/sdlc-docs` produces evidence base

## Inputs

- **Repo path** (default: cwd)
- **Evidence manifest** (optional: `docs/sdlc/manifest.json` from `sdlc-docs`) — if absent, skill collects evidence itself but warns user that `/sdlc-docs` first gives richer attestation
- **Org context** (optional): vendor name, software product name, point-of-contact — fills attestation header. Prompt if missing.

## Execution flow

### Phase 1 — Evidence collection

If `manifest.json` present, load and use as primary evidence. Else gather inline:

| Practice area | Evidence sources |
|---------------|------------------|
| **Source integrity** | Branch protection rules, signed commits config, CODEOWNERS, required reviewers |
| **Build integrity** | CI configs, SLSA provenance attestations, reproducible build flags, artifact signing (cosign/sigstore) |
| **Dependency mgmt** | Lockfiles present, dependabot/renovate, SBOM workflow, pinned versions |
| **Vuln mgmt** | SECURITY.md, disclosure policy, SLA, patching cadence in CI |
| **Secrets** | gitleaks/trufflehog, GitHub secret scanning, vault references in code (no inline secrets) |
| **SAST/DAST** | CodeQL/Semgrep/Bandit/SonarQube configs, DAST in staging pipeline |
| **Testing** | Coverage threshold gates, security test suite, fuzzing |
| **Logging/monitoring** | Structured logging library use, log redaction, SIEM forwarder config |
| **Access control** | IaC for IAM, MFA enforcement on repo, least-privilege docs |
| **Incident response** | Runbook present, on-call doc, postmortem template |

Commands to run (read-only, safe):

```bash
git log --show-signature -1
git log --pretty=format:'%G?' -10 | sort -u    # Signature status histogram
gh api repos/{owner}/{repo}/branches/{branch}/protection 2>/dev/null
gh api repos/{owner}/{repo}/vulnerability-alerts 2>/dev/null
gh api repos/{owner}/{repo}/code-scanning/alerts 2>/dev/null
gh api repos/{owner}/{repo}/secret-scanning/alerts 2>/dev/null
gh api repos/{owner}/{repo}/dependabot/alerts 2>/dev/null
find . -maxdepth 4 -type f \( -name "SECURITY.md" -o -name "CODEOWNERS" -o -name "dependabot.yml" -o -name "*.semgrep*" -o -name ".gitleaks*" -o -name "codeql*.yml" \)
```

### Phase 2 — Map to frameworks

For each control in each framework, assign:

- **Status**: `met` | `partial` | `gap` | `not_applicable`
- **Confidence**: `high` | `medium` | `low` (based on evidence directness)
- **Evidence**: file path, command output snippet, or `none`
- **Notes**: 1-2 line rationale

#### NIST SSDF mapping (full practice set)

```
PO.1  Define security requirements         — docs/sdlc/00-overview.md, SECURITY.md
PO.2  Implement roles & responsibilities   — CODEOWNERS, org chart
PO.3  Implement supporting toolchains      — CI workflows, secrets scanner
PO.4  Define & use criteria for sw security — branch protection, required checks
PO.5  Implement secure environments        — IaC, secrets manager
PS.1  Protect all forms of code from unauth — branch protection, signed commits, MFA
PS.2  Provide mechanism for verifying integrity — release signing, checksums
PS.3  Archive & protect each release       — release artifacts retained, immutable storage
PW.1  Design to meet security requirements — threat model present
PW.2  Review design                        — ADRs / design review records
PW.4  Reuse existing well-secured software — vetted deps, SBOM
PW.5  Create source code adhering to secure — secure-coding standard, linter rules
PW.6  Configure compilation/build/interp   — hardening flags, deterministic builds
PW.7  Review/analyze human-readable code   — code review enforcement, SAST
PW.8  Test executable code                 — DAST, fuzzing, security tests
PW.9  Configure software with secure defaults — config-as-code reviewed
RV.1  Identify & confirm vulns on ongoing  — dep alerts enabled, scanning cadence
RV.2  Assess, prioritize & remediate vulns — SLA per CVSS, tracking issues
RV.3  Analyze vulns to identify root causes — postmortem doc, recurring trend log
```

#### ISM + Essential Eight mapping

Score Essential Eight maturity per mitigation (ML0–ML3):

```
E8-1  Application control
E8-2  Patch applications
E8-3  Configure MS Office macro settings   (often N/A for code repos — mark NA)
E8-4  User application hardening
E8-5  Restrict administrative privileges
E8-6  Patch operating systems
E8-7  Multi-factor authentication
E8-8  Regular backups
```

ISM controls to assess (subset relevant to software dev):
ISM-0072 (secret scanning), ISM-0670 (vuln scanning), ISM-1238 (SAST), ISM-1419 (secure coding), ISM-1467 (dep mgmt), ISM-1616 (logging), ISM-1730 (SBOM), ISM-1819 (branch protection), ISM-1872 (signed commits), ISM-1894 (SLSA provenance).

#### OWASP SAMM v2 (5 functions × 3 practices, score 0-3)

- Governance: Strategy & Metrics, Policy & Compliance, Education & Guidance
- Design: Threat Assessment, Security Requirements, Security Architecture
- Implementation: Secure Build, Secure Deployment, Defect Management
- Verification: Architecture Assessment, Requirements-driven Testing, Security Testing
- Operations: Incident Management, Environment Management, Operational Management

#### ASVS v4 L2 — verification requirements (chapters V1–V14)

Score each chapter `pass` | `partial` | `fail` | `not_tested` with chapter-level summary.

### Phase 3 — Compute posture score

Overall posture: weighted blend.

```
posture_score = 0.4 * ssdf_pct + 0.3 * (e8_avg_ml / 3) + 0.2 * (samm_avg / 3) + 0.1 * asvs_pct
```

Bands: `≥0.85 Strong` · `0.65–0.84 Adequate` · `0.40–0.64 Developing` · `<0.40 Inadequate`.

### Phase 4 — Emit outputs

Write to `docs/attestation/`:

- `attestation-report-<YYYY-MM-DD>.md` — human report (see template)
- `attestation-<YYYY-MM-DD>.json` — machine-readable
- `gaps.json` — gap list, consumed by `ssdf-remediate`

### Markdown report template

```markdown
---
generated_at: <ISO8601>
generator: ssdf-attest
repo: <path>
head: <sha>
frameworks: [NIST-SSDF-v1.1, ISM, E8, SAMM-v2, ASVS-v4-L2]
posture_score: 0.72
posture_band: Adequate
---

# Secure Software Development Attestation

## 1. Attestation header
- **Software / product**: <name>
- **Producer / vendor**: <org>
- **Point of contact**: <email>
- **Version / commit**: <sha>
- **Assessment date**: <date>
- **Scope**: repository <path>; <inclusions/exclusions>
- **Frameworks**: NIST SSDF v1.1, Australian ISM + Essential Eight, OWASP SAMM v2, OWASP ASVS v4 L2

## 2. Executive summary
- Overall posture: **Adequate (0.72)**
- 12 controls met · 7 partial · 4 gaps · 2 N/A
- Top 3 gaps: <list>
- Highest-risk gap: <one-liner>

## 3. NIST SSDF v1.1 results
<table per practice with status, evidence, notes>

## 4. Australian ISM + Essential Eight
<E8 maturity radar + ISM control table>

## 5. OWASP SAMM v2 maturity
<5 function × 3 practice scores>

## 6. OWASP ASVS v4 L2
<chapter pass/fail summary>

## 7. Gap register
<table: ID, framework, control, severity, current state, required state, owner, remediation summary>

## 8. Attestation statement
Based on the evidence collected on <date>, the <product> codebase is assessed at **<band>** posture against the listed frameworks. **This automated assessment is advisory and is not a substitute for an independent audit, IRAP assessment, or signed CISA Common Form.** Where the score is below `Strong`, see §7 for the remediation plan. A remediation prompt for automated fixes is provided in `gaps.json` and can be applied via `/ssdf-remediate`.

## 9. Recommended next step
Run `/ssdf-remediate docs/attestation/gaps.json` to generate a prioritised AI agent prompt that applies the fixes. Re-run `/ssdf-attest` after remediation to verify uplift.
```

### gaps.json schema

```json
{
  "generated_at": "<ISO>",
  "repo": "<path>",
  "posture_score": 0.72,
  "gaps": [
    {
      "id": "GAP-001",
      "framework": "NIST-SSDF",
      "control": "PW.7",
      "title": "No SAST in CI",
      "severity": "high",
      "current": "No CodeQL/Semgrep workflow detected",
      "required": "SAST runs on every PR with blocking thresholds on high/critical",
      "evidence_path": ".github/workflows/",
      "remediation_hint": "Add CodeQL workflow targeting <language>; gate merge on high+ findings",
      "estimated_effort": "small"
    }
  ]
}
```

## Rules

- **No fabrication**: every status backed by file path, command output, or explicit `none`. If evidence cannot be retrieved (private API, missing tool), mark confidence `low` and note assumption.
- **Advisory only**: report must include the §8 disclaimer. This is not a signed legal attestation.
- **Privacy**: do not include secret values, tokens, or PII in the report. Redact paths that contain account IDs unless user opts in.
- **Reproducibility**: every command run logged in an appendix `commands.log` so an auditor can rerun.
- **Caveman mode**: report itself is formal English. Only chat output to user uses caveman.
- **Hand-off**: end-of-run, tell user: posture band, gap count, and `Run /ssdf-remediate docs/attestation/gaps.json next`.

## Completion criteria

- `attestation-report-<date>.md` written
- `attestation-<date>.json` written
- `gaps.json` written
- Console summary: posture band, score, gap count, top-3 gap IDs
- Next-step instruction emitted
