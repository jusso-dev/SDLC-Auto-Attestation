---
name: sdlc-docs
description: Generate full SDLC documentation pack for a code repository. Produces design docs, threat model, risk register, SBOM notes, test strategy, deployment runbook, change management, and security controls catalogue. Use when user asks to "generate SDLC docs", "document this project", "produce SDLC pack", or before running ssdf-attest. Aligned with NIST SSDF v1.1, OWASP SAMM, and Australian ISM.
argument-hint: "[repo path or --out <dir>]"
---

# SDLC Documentation Generator

Generates a complete Software Development Lifecycle documentation pack for any code repository. Output is the evidence base that `ssdf-attest` consumes for attestation.

## When to invoke

- User points skill at a repo and asks for SDLC documentation
- User runs `/sdlc-docs <path>`
- Pre-requisite step before `ssdf-attest` if repo lacks docs
- User says: "document this project", "generate SDLC pack", "produce design docs"

## Inputs

- **Repo path** (default: cwd)
- **Output dir** (default: `<repo>/docs/sdlc/`)
- **Project metadata** (auto-detect; prompt only if ambiguous): name, owner, classification (public/internal/restricted), regulatory scope

## Execution flow

### Phase 1 — Discover

Run in parallel where possible. Do NOT read every file; sample strategically.

1. **Repo shape**: `git log --oneline -20`, `git remote -v`, default branch, branch protection (if `gh` available: `gh api repos/{owner}/{repo}/branches/{default}/protection`)
2. **Language & stack**: detect from manifest files (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`, `requirements.txt`, `Gemfile`, `composer.json`)
3. **Build & test**: CI configs (`.github/workflows/*`, `.gitlab-ci.yml`, `Jenkinsfile`, `.circleci/`, `azure-pipelines.yml`)
4. **Deployment**: `Dockerfile`, `docker-compose*.yml`, `k8s/`, `terraform/`, `serverless.yml`, `*.tf`, `helm/`, `Procfile`, `fly.toml`, `vercel.json`
5. **Dependencies**: lockfiles, `SECURITY.md`, dependabot/renovate configs
6. **Security tooling**: SAST/DAST/secrets-scan configs (`.semgrep.yml`, `trivy.yaml`, `.gitleaks.toml`, `bandit.yaml`, CodeQL workflow)
7. **Existing docs**: `README*`, `CONTRIBUTING*`, `SECURITY*`, `ARCHITECTURE*`, `docs/**`, `ADR*`
8. **Entry points / surfaces**: HTTP routes, RPC handlers, CLI entrypoints, background workers, public packages
9. **Data**: schemas (`schema.prisma`, `*.sql`, `migrations/`), env vars (`.env.example`), secret references

Use `codegraph_search` / `codegraph_files` if `.codegraph/` exists. Do NOT use `codegraph_explore` in main session (returns too much content).

### Phase 2 — Generate

Write all docs under output dir. Each doc has YAML frontmatter (`generated_at`, `generator: sdlc-docs`, `repo_sha`).

```
docs/sdlc/
├── 00-overview.md              # Project charter, scope, owners, classification
├── 01-architecture.md          # C4 L1+L2 diagrams (Mermaid), tech stack, key dependencies
├── 02-data-flow.md             # Data classification, flows, storage, retention, sovereignty
├── 03-threat-model.md          # STRIDE per component + trust boundaries + mitigations
├── 04-risk-register.md         # Tabular: ID, risk, likelihood, impact, owner, treatment, residual
├── 05-security-controls.md     # Controls catalogue mapped to NIST SSDF + ISM E8 + ASVS L2
├── 06-secure-coding.md         # Standards, language rules, mandatory linters/SAST, review gates
├── 07-test-strategy.md         # Unit/integration/e2e coverage targets, security testing, perf
├── 08-build-release.md         # Build reproducibility, provenance (SLSA), signing, SBOM gen
├── 09-deployment-runbook.md    # Environments, promotion, rollback, feature flags, canaries
├── 10-incident-response.md     # On-call, severity matrix, comms, post-mortems
├── 11-change-management.md     # PR policy, branch protection, approvals, emergency change
├── 12-access-control.md        # RBAC, secrets mgmt, key rotation, identity providers
├── 13-vuln-mgmt.md             # Disclosure policy, SLA per severity, dependency patching cadence
├── 14-data-protection.md       # Encryption at rest/transit, PII handling, backups, DR (RPO/RTO)
├── 15-sbom.md                  # SBOM generation command + last SBOM summary (CycloneDX/SPDX)
└── manifest.json               # Index + checksums for ssdf-attest to consume
```

### Phase 3 — Manifest

Emit `manifest.json` listing every doc, its source-of-truth (file paths it summarises), and a `gaps[]` array for sections that could not be filled from repo evidence.

```json
{
  "generated_at": "2026-05-14T00:00:00Z",
  "repo": {"path": "...", "head": "<sha>", "remote": "..."},
  "docs": [{"id": "03-threat-model", "path": "...", "evidence": ["src/api/*"], "completeness": 0.6}],
  "gaps": [{"doc": "10-incident-response", "missing": "on-call rotation, sev matrix"}]
}
```

## Document templates

### 00-overview.md

```markdown
# Project Overview
- **Name**: <name>
- **Owner**: <team / individual>
- **Classification**: Public | Internal | Restricted | Protected
- **Regulatory scope**: e.g. PCI-DSS, APRA CPS 234, GDPR, HIPAA, IRAP
- **Purpose**: 1-2 sentences
- **Users / actors**: who uses it
- **Critical assets**: data, IP, availability targets
- **SLAs**: availability, RPO, RTO
```

### 03-threat-model.md (STRIDE)

For each component identified in architecture, table:

| Threat | STRIDE | Likelihood | Impact | Mitigation | Status |
|--------|--------|-----------|--------|------------|--------|

Plus trust boundary diagram (Mermaid).

### 05-security-controls.md

Controls table mapped to frameworks:

| Control | NIST SSDF | ISM | ASVS | Implementation | Evidence path |
|---------|-----------|-----|------|----------------|---------------|
| Branch protection | PO.3.2, PW.7 | ISM-1819 | V14.2 | GitHub required reviews | `.github/CODEOWNERS` |
| SAST | PW.7, PW.8 | ISM-1238 | V14.3 | CodeQL workflow | `.github/workflows/codeql.yml` |
| Dep scanning | PW.4, RV.1 | ISM-1467 | V14.2 | Dependabot weekly | `.github/dependabot.yml` |
| Secret scanning | PS.1, PW.7 | ISM-0072 | V2.10 | gitleaks pre-commit | `.gitleaks.toml` |
| SBOM | PS.3, PW.4 | ISM-1730 | — | CycloneDX in CI | `<workflow>` |

### 15-sbom.md

Include generation commands (don't run blindly — note for user):

```bash
# Node
npx @cyclonedx/cyclonedx-npm --output-file sbom.cdx.json
# Python
cyclonedx-py -o sbom.cdx.json
# Go
cyclonedx-gomod mod -json -output sbom.cdx.json
# Container
syft <image> -o cyclonedx-json > sbom.cdx.json
```

## Rules

- **Evidence-anchored**: every claim cites a file path or command output. If evidence missing, mark `[GAP]` and log in `manifest.json.gaps[]` — do NOT fabricate.
- **No invented controls**: if SAST config absent, write "No SAST configured" not "SAST runs via X".
- **Mermaid for diagrams**: keep diagrams in markdown — no external image files.
- **Frontmatter on every doc**: enables downstream parsing.
- **Idempotent**: re-running overwrites cleanly. Existing hand-edits in non-generated sections preserved if user marks `<!-- preserve -->` blocks.
- **Caveman mode**: docs themselves are professional prose, not caveman style. Caveman applies only to user-facing chat.

## Completion criteria

- All 16 files written
- `manifest.json` lists every doc with `completeness` score 0-1
- Console output: summary table of completeness + gap count
- Tell user: "Run `/ssdf-attest` next to assess posture against frameworks."
