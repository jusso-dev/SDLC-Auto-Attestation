---
name: ssdf-remediate
description: Consume a gaps.json file produced by ssdf-attest and emit a structured AI-agent remediation prompt that, when fed to a coding agent, applies the fixes to close attestation gaps before re-running ssdf-attest. Use when user runs "/ssdf-remediate", says "fix the gaps", "generate remediation prompt", "close attestation findings", or after ssdf-attest reports posture below Strong.
argument-hint: "[path to gaps.json — default docs/attestation/gaps.json]"
---

# Attestation Gap Remediation Prompt Generator

Produces a deterministic, AI-agent-executable prompt that fixes the gaps surfaced by `ssdf-attest`. The output is a single self-contained markdown prompt the user can paste into Claude Code, Codex, Cursor, or any agentic IDE — or this skill can dispatch it directly via the `Agent` tool if the user agrees.

## When to invoke

- Immediately after `ssdf-attest` if posture < `Strong`
- User runs `/ssdf-remediate [gaps.json]`
- User says: "fix the gaps", "remediate attestation", "generate fix prompt", "close findings"

## Inputs

- **gaps.json path** (default `docs/attestation/gaps.json`)
- **Mode** (prompt user once):
  - `prompt-only` — emit prompt file only (default, safe)
  - `dispatch` — also spawn an `Agent` subagent to apply the fixes in an isolated worktree
- **Severity floor** (optional, default `medium`): skip gaps below this severity
- **Exclude controls** (optional): list of control IDs to skip (e.g. user declines macros control for non-Office product)

## Execution flow

### Phase 1 — Load & validate gaps

1. Read `gaps.json`. Fail loudly if schema invalid.
2. Filter by severity floor and exclude list.
3. Group gaps by remediation type:
   - **CI/workflow additions** (SAST, dep scan, SBOM gen, signing)
   - **Repo config** (branch protection, CODEOWNERS, SECURITY.md, dependabot.yml)
   - **Code changes** (input validation, crypto upgrades, logging redaction)
   - **Docs additions** (threat model, runbook, IR plan)
   - **Process changes** (cannot auto-fix — flag for human, e.g. MFA enforcement, vendor reviews)

### Phase 2 — Prioritise

Order by: severity (critical > high > medium > low), then effort (small first within severity), then dependency order (e.g. add SBOM workflow before signing workflow that consumes it).

### Phase 3 — Emit prompt

Write `docs/attestation/remediation-prompt-<YYYY-MM-DD>.md`. Format:

```markdown
# Attestation Gap Remediation — Agent Prompt

## Mission
You are a security engineer agent. Close the attestation gaps listed below in the repository at `<repo path>`. After each gap, verify the fix locally where possible. Do not invent files outside the listed paths. Open one PR per logical group (CI / config / code / docs).

## Ground rules
1. Work in a feature branch `chore/ssdf-remediation-<date>`.
2. Make minimal, reviewable changes — no drive-by refactors.
3. Where a gap requires a process/human decision (e.g. enforcing org-level MFA), STOP and emit a TODO line in `REMEDIATION-TODO.md` instead of forging evidence.
4. Preserve existing CI behaviour — add jobs, do not replace.
5. After all gaps closed, run the project's existing test suite. Report failures.
6. Do NOT modify production secrets or rotate keys. Flag any such need.
7. Commit message format: `chore(ssdf): close <GAP-ID> — <control>` per gap. Sign commits if signing is configured.

## Gap backlog (prioritised)

### GAP-001 — <title>   [framework · control · severity]
- **Current state**: <current>
- **Required state**: <required>
- **Evidence path**: <path>
- **Action**: <concrete step-by-step>
- **Acceptance criteria**: <how to verify; e.g. "CodeQL workflow exists at .github/workflows/codeql.yml and runs on PR">
- **Reference**: <NIST SSDF section / ISM control / ASVS requirement>

<repeat per gap>

## Verification step
After all gaps applied:
1. Run `git status` — confirm only expected files changed.
2. Run any linters/tests touched by the changes.
3. Generate a fresh SBOM if SBOM tooling was added.
4. Stage and commit per the grouping above.
5. Output a summary table: `GAP-ID | status (done/blocked) | files changed | notes`.

## Re-attestation
After PR(s) merged or branch pushed, the user will re-run `/ssdf-attest` to verify posture uplift. Do NOT run attestation yourself — that is the user's verification gate.
```

### Phase 4 — Optional dispatch

If user chose `dispatch` mode:

1. Confirm with user — show the prompt summary first (count of gaps, severities, est effort).
2. Use the `Agent` tool with `subagent_type: general-purpose` and `isolation: worktree` so the work happens on an isolated copy.
3. Pass the entire emitted prompt file as the agent prompt.
4. Surface the agent's summary and worktree path back to the user. Do not auto-merge.

## Auto-fix recipe library

Skill ships with deterministic templates for common gaps. When a gap matches a known recipe, embed the literal patch in the prompt (so the downstream agent doesn't have to guess).

### Recipe: add CodeQL SAST (NIST PW.7, ISM-1238)

```yaml
# .github/workflows/codeql.yml
name: CodeQL
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
  schedule: [{ cron: "0 6 * * 1" }]
permissions:
  security-events: write
  contents: read
  actions: read
jobs:
  analyze:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        language: [<detect>]
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with: { languages: ${{ matrix.language }} }
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
```

### Recipe: gitleaks secret scan (NIST PS.1, ISM-0072)

```yaml
# .github/workflows/gitleaks.yml
name: gitleaks
on: [push, pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v2
```

### Recipe: dependabot enable (NIST PW.4, ISM-1467)

```yaml
# .github/dependabot.yml — auto-detect ecosystems
version: 2
updates:
  - package-ecosystem: "<npm|pip|gomod|cargo|maven>"
    directory: "/"
    schedule: { interval: "weekly" }
    open-pull-requests-limit: 10
    labels: ["dependencies", "security"]
```

### Recipe: CycloneDX SBOM (NIST PS.3, ISM-1730)

```yaml
# .github/workflows/sbom.yml
name: SBOM
on:
  release: { types: [published] }
  workflow_dispatch:
jobs:
  sbom:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Generate SBOM
        run: <ecosystem-specific cyclonedx command>
      - uses: actions/upload-artifact@v4
        with: { name: sbom, path: sbom.cdx.json }
```

### Recipe: sigstore/cosign artifact signing (NIST PS.2, ISM-1894)

```yaml
# .github/workflows/sign.yml
name: Sign release
on:
  release: { types: [published] }
permissions:
  id-token: write
  contents: read
jobs:
  sign:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: sigstore/cosign-installer@v3
      - run: cosign sign-blob --yes <artifact>
```

### Recipe: SECURITY.md (NIST RV.1, ISM-1616)

```markdown
# Security Policy
## Reporting a vulnerability
Email: security@<domain>  (or use GitHub private vulnerability reporting)
We acknowledge within 2 business days; triage within 5; fix SLA based on CVSS:
- Critical: 7d · High: 30d · Medium: 90d · Low: best-effort
## Supported versions
<table>
```

### Recipe: CODEOWNERS for required reviews (NIST PO.2, ISM-1819)

```
# .github/CODEOWNERS
*               @<org>/maintainers
/src/auth/**    @<org>/security
/.github/**     @<org>/platform
```

### Recipe: branch protection (manual, gh CLI)

```bash
gh api -X PUT repos/{owner}/{repo}/branches/main/protection \
  -F required_status_checks.strict=true \
  -F required_status_checks.contexts[]=CodeQL \
  -F enforce_admins=true \
  -F required_pull_request_reviews.required_approving_review_count=2 \
  -F required_pull_request_reviews.require_code_owner_reviews=true \
  -F required_linear_history=true \
  -F restrictions=null
```

## Rules

- **No silent escalation**: changes that need human approval (branch protection, MFA, key rotation, vendor onboarding) get added to `REMEDIATION-TODO.md` — never auto-applied.
- **Idempotent**: re-running with same `gaps.json` produces the same prompt.
- **Traceability**: every action in the prompt cites its GAP-ID and the framework control it closes.
- **Worktree isolation**: dispatch mode always uses `isolation: worktree` so the user can diff before merging.
- **Verification gate**: skill does NOT re-run attestation itself. The loop is: attest → remediate → user reviews → user re-runs attest.
- **Caveman mode**: user-facing chat is caveman; the emitted prompt file is formal English (the downstream agent reads it).

## Completion criteria

- `remediation-prompt-<date>.md` written
- If dispatch mode: agent spawned in worktree, agent summary surfaced, worktree path reported
- Console summary: gap count, prompt path, next-step ("review prompt then dispatch or paste into agent of choice")
- Final instruction: "After fixes applied & merged, re-run `/ssdf-attest` to confirm posture uplift."
