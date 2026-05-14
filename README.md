# Secure SDLC Skills for Claude Code

A trio of Claude Code skills that point at any code repository and produce, in order: a full SDLC documentation pack, a multi-framework cyber-posture attestation report, and an AI-agent remediation prompt that closes the gaps before the next attestation run.

> **TL;DR** — Run `/sdlc-docs` → `/ssdf-attest` → `/ssdf-remediate` on any repo. You get audit-ready documentation, a scored attestation against NIST SSDF + Australian ISM/Essential Eight + OWASP SAMM/ASVS, and an executable prompt that an AI agent uses to fix what's missing. Then re-attest to prove uplift.

---

## What is a Claude skill?

A **Claude skill** is a self-contained instruction file (`SKILL.md`) that lives in `~/.claude/skills/<name>/` and teaches Claude Code how to perform a specific task end-to-end. Each skill has YAML frontmatter declaring its `name` and `description`; Claude inspects descriptions at session start and auto-invokes the right skill when the user's request matches, or the user can trigger one explicitly with `/skill-name`.

Skills are different from prompts and from agents:

| | Prompt | Skill | Subagent |
|--|--|--|--|
| **Where it lives** | Inline message | `~/.claude/skills/<name>/SKILL.md` | `~/.claude/agents/<name>.md` |
| **How invoked** | User types it each time | `/name` or auto-trigger via description match | Spawned by Claude with its own context window |
| **Scope** | One-shot | Reusable procedure with rules, recipes, templates | Isolated worker for parallel/long tasks |
| **State** | None | Reads/writes repo files | Returns a single summary to caller |

Think of a skill as a checked-in playbook the model loads on demand — same shape every time, no copy-paste, no drift.

The skills in this package are installed globally at `~/.claude/skills/` so they work from any project directory.

---

## Why you might want to use this

If you ship software into any of these contexts, an attestation of secure development practices is either already required or about to be:

- **Selling to US Federal**: M-22-18 + CISA Secure Software Development Attestation Common Form requires producers to attest alignment with NIST SP 800-218 (SSDF).
- **Selling to Australian Government**: ISM controls + Essential Eight maturity are mandatory for non-corporate Commonwealth entities and increasingly demanded by State agencies and regulated industries (APRA CPS 234, ISO 27001 mappings).
- **Regulated industries** (banking, health, critical infrastructure): SOCI Act, APRA CPS 234, HIPAA, PCI-DSS all expect documented secure-SDLC practices.
- **Enterprise procurement**: vendor security questionnaires routinely ask for SBOMs, SDLC evidence, threat models, IR plans.
- **Open source maintainers** chasing OpenSSF Scorecard, SLSA levels, or CNCF graduation criteria.

Even outside compliance, the same artefacts answer practical questions: *Do we have a threat model? Where are our secrets scanned? What is our patching SLA? Can a new engineer find the deployment runbook?*

Producing this documentation by hand takes weeks. These skills produce a defensible first draft in minutes, anchored to actual evidence in the repo — no fabricated controls.

### What you get from one full pipeline run

1. **`docs/sdlc/`** — 16 SDLC documents: overview, C4 architecture, data flow, STRIDE threat model, risk register, controls catalogue, secure-coding standard, test strategy, build/release, deployment runbook, IR plan, change management, access control, vuln management, data protection, SBOM. Plus a `manifest.json` with completeness scores.
2. **`docs/attestation/`** — Markdown attestation report + machine-readable JSON + `gaps.json`, scored against NIST SSDF v1.1, Australian ISM + Essential Eight maturity, OWASP SAMM v2, OWASP ASVS v4 L2. Posture band: Strong / Adequate / Developing / Inadequate.
3. **`docs/attestation/remediation-prompt-<date>.md`** — A self-contained AI-agent prompt that, when dispatched to Claude Code / Codex / Cursor, applies the fixes to close the gaps. Optionally auto-dispatched into an isolated git worktree.

### What this is *not*

- **Not a signed legal attestation.** The CISA Common Form requires a company CEO or designee to sign under penalty of false statements. This tool produces the *evidence and assessment* that supports that signature; it does not replace it.
- **Not an IRAP assessment.** Australian Government PROTECTED-level work requires a registered IRAP assessor.
- **Not a substitute for penetration testing.** SAST/dep scanning ≠ adversarial testing.
- **Not a substitute for governance.** It cannot enforce MFA, rotate keys, sign contracts, or get your CISO to approve a risk treatment.

Treat the output as the world's most thorough first draft, reviewed by humans before any external use.

---

## Frameworks covered

| Framework | What it is | What this tool checks |
|-----------|------------|------------------------|
| **NIST SSDF v1.1 (SP 800-218)** | US federal Secure Software Development Framework. Practices grouped as Prepare the Organisation (PO), Protect the Software (PS), Produce Well-Secured Software (PW), Respond to Vulnerabilities (RV). | All 19 practices — branch protection, signed commits, SAST, threat modelling, SBOM, vulnerability disclosure, etc. |
| **Australian ISM** | ACSC Information Security Manual — control catalogue used across Commonwealth, State, and regulated entities. | Subset of controls relevant to software development: ISM-0072, 0670, 1238, 1419, 1467, 1616, 1730, 1819, 1872, 1894 (extensible). |
| **Essential Eight** | ACSC's prioritised mitigation strategies, scored at Maturity Levels 0–3. | E8 strategies 1, 2, 4, 5, 6, 7, 8 (E8-3 macros is typically N/A for code repos). |
| **OWASP SAMM v2** | Software Assurance Maturity Model — 5 business functions × 3 practices, scored 0–3. | All 15 practices across Governance / Design / Implementation / Verification / Operations. |
| **OWASP ASVS v4 L2** | Application Security Verification Standard, Level 2 (most apps handling sensitive data). | All 14 chapters with chapter-level pass/partial/fail/not-tested status. |

Output reports the score in each framework plus an overall weighted posture score.

---

## Install

Already installed if you can see all three skills in your Claude Code skills list. If not, clone this repo and copy (or symlink) the skill directories into `~/.claude/skills/`:

```bash
git clone https://github.com/jusso-dev/SDLC-Auto-Attestation.git
cd SDLC-Auto-Attestation
mkdir -p ~/.claude/skills
cp -R skills/sdlc-docs skills/ssdf-attest skills/ssdf-remediate ~/.claude/skills/
# Or symlink instead of copy, to track upstream changes:
#   for s in sdlc-docs ssdf-attest ssdf-remediate; do
#     ln -s "$(pwd)/skills/$s" ~/.claude/skills/$s
#   done
```

Verify:

```bash
ls ~/.claude/skills/*/SKILL.md
# Should list:
#   ~/.claude/skills/sdlc-docs/SKILL.md
#   ~/.claude/skills/ssdf-attest/SKILL.md
#   ~/.claude/skills/ssdf-remediate/SKILL.md
```

Start a new Claude Code session in the target repository — the skills will appear in the available-skills list automatically.

### Optional prerequisites

The skills degrade gracefully if these are missing, but richer output if you have them:

- **`gh` CLI**, authenticated against your GitHub org — unlocks branch-protection, code-scanning, secret-scanning, and dependabot alert evidence.
- **CodeGraph** initialised (`.codegraph/` in the repo) — speeds up code discovery during `sdlc-docs`.
- **CycloneDX or Syft** installed locally — needed only if you want to generate an SBOM during remediation rather than in CI.

---

## Usage

### Full pipeline (recommended)

```text
# In Claude Code, inside the target repo:
/sdlc-docs           # build evidence base under docs/sdlc/
/ssdf-attest         # score against all frameworks; writes docs/attestation/
/ssdf-remediate      # generate fix prompt from gaps.json
# Review the prompt, dispatch it (or paste into your coding agent of choice).
# After fixes are merged:
/ssdf-attest         # re-score; confirm posture uplift
```

### Just the documentation pack

```text
/sdlc-docs
# Reads the repo, writes docs/sdlc/*.md and manifest.json. No scoring.
```

### Just the attestation (skips doc generation)

```text
/ssdf-attest
# Collects evidence inline. Less rich than running /sdlc-docs first,
# but works on repos where you don't want generated docs committed.
```

### Auto-dispatch remediation into a worktree

```text
/ssdf-remediate --mode=dispatch
# Spawns a subagent in an isolated git worktree, applies the recipes,
# returns the worktree path. You diff and merge before re-attesting.
```

### Argument summary

| Skill | Args | Default |
|-------|------|---------|
| `sdlc-docs` | `[repo path] [--out <dir>]` | cwd → `docs/sdlc/` |
| `ssdf-attest` | `[repo path] [--evidence <manifest.json>]` | cwd → `docs/attestation/` |
| `ssdf-remediate` | `[gaps.json path] [--mode prompt-only\|dispatch] [--severity-floor <s>]` | `docs/attestation/gaps.json`, `prompt-only`, `medium` |

---

## The loop, in detail

```
                ┌────────────────────────────────────────┐
                │  Repository                            │
                │  (any language, any stack)             │
                └─────────────┬──────────────────────────┘
                              │
                              ▼
         ┌────────────────────────────────────────┐
         │  /sdlc-docs                            │
         │  → docs/sdlc/00-overview.md … 15-sbom  │
         │  → manifest.json (completeness + gaps) │
         └─────────────┬──────────────────────────┘
                       │
                       ▼
         ┌────────────────────────────────────────┐
         │  /ssdf-attest                          │
         │  → attestation-<date>.md               │
         │  → attestation-<date>.json             │
         │  → gaps.json                           │
         │  Posture: Strong | Adequate |          │
         │          Developing | Inadequate       │
         └─────────────┬──────────────────────────┘
                       │  posture < Strong?
                       ▼
         ┌────────────────────────────────────────┐
         │  /ssdf-remediate                       │
         │  → remediation-prompt-<date>.md        │
         │  (or dispatched to Agent in worktree)  │
         └─────────────┬──────────────────────────┘
                       │  human review + merge
                       ▼
                  /ssdf-attest  (re-score, confirm uplift)
```

---

## Design principles

The skills enforce these rules so the output is defensible:

1. **No fabrication.** Every status maps to a file path, command output, or an explicit `none`. If a control's evidence cannot be retrieved, confidence is marked `low` and the assumption is logged — never invented.
2. **Advisory disclaimer.** Every attestation report carries §8 — automated assessment, not a substitute for audit/IRAP/signed Common Form.
3. **Privacy.** Secret values, tokens, and PII are redacted from outputs. Account-ID-bearing paths require opt-in.
4. **Idempotent.** Re-running on the same repo state produces the same output (modulo timestamps and `repo_sha`).
5. **Traceability.** Every gap-fix in the remediation prompt cites the GAP-ID and the control it closes; every command run by attestation is logged to `commands.log` for auditor reproduction.
6. **Human-gated changes.** Anything that needs a human decision (branch protection, MFA enforcement, key rotation, vendor onboarding, signing PRs) is flagged to `REMEDIATION-TODO.md`. The remediation agent never auto-applies these.
7. **Worktree isolation.** Dispatched remediation runs in an isolated git worktree; the user diffs and merges. No direct writes to the working branch.
8. **Re-attestation is user-gated.** The pipeline does not auto-pass its own audit — the user re-runs `/ssdf-attest` after reviewing remediations.

---

## Output layout

```
<repo>/
├── docs/
│   ├── sdlc/
│   │   ├── 00-overview.md
│   │   ├── 01-architecture.md
│   │   ├── 02-data-flow.md
│   │   ├── 03-threat-model.md
│   │   ├── 04-risk-register.md
│   │   ├── 05-security-controls.md
│   │   ├── 06-secure-coding.md
│   │   ├── 07-test-strategy.md
│   │   ├── 08-build-release.md
│   │   ├── 09-deployment-runbook.md
│   │   ├── 10-incident-response.md
│   │   ├── 11-change-management.md
│   │   ├── 12-access-control.md
│   │   ├── 13-vuln-mgmt.md
│   │   ├── 14-data-protection.md
│   │   ├── 15-sbom.md
│   │   └── manifest.json
│   └── attestation/
│       ├── attestation-report-<date>.md
│       ├── attestation-<date>.json
│       ├── gaps.json
│       ├── remediation-prompt-<date>.md
│       ├── REMEDIATION-TODO.md       # human-gated items
│       └── commands.log              # full audit trail
```

Recommended `.gitignore` policy: commit `docs/sdlc/` (it's evidence), commit `docs/attestation/*.md` (the report), do **not** commit `commands.log` if it contains org/account identifiers.

---

## Built-in remediation recipes

`ssdf-remediate` ships with deterministic patches for the most common gaps — these are embedded verbatim in the prompt so the downstream agent does not have to guess:

- **CodeQL SAST** workflow (NIST PW.7, ISM-1238)
- **gitleaks** secret scan (NIST PS.1, ISM-0072)
- **Dependabot** dependency updates (NIST PW.4, ISM-1467)
- **CycloneDX SBOM** generation on release (NIST PS.3, ISM-1730)
- **Sigstore / cosign** release signing (NIST PS.2, ISM-1894)
- **SECURITY.md** vulnerability disclosure policy (NIST RV.1, ISM-1616)
- **CODEOWNERS** for required reviews (NIST PO.2, ISM-1819)
- **Branch protection** (gh CLI commands, human-gated)

Other gaps get tailored instructions generated from the gap descriptor.

---

## FAQ

**Q: Does this work on monorepos?**
Yes. Point `/sdlc-docs` at the repo root or a sub-package. The skill detects multiple manifest files and documents the relevant boundary.

**Q: Does it require GitHub?**
No, but evidence is richer on GitHub because of the API endpoints (branch protection, code-scanning alerts, dependabot). Works on GitLab/Bitbucket/local-only — falls back to file-system evidence with confidence noted as lower.

**Q: Can I add controls to the catalogue?**
Yes. The control mapping lives in the skill files themselves — edit `~/.claude/skills/ssdf-attest/SKILL.md` and add rows to the relevant framework table.

**Q: Will this leak my code to Anthropic?**
The skills run inside your existing Claude Code session — they obey whatever data-handling agreement you already have with Anthropic. The skills themselves do not call out to any third party.

**Q: Does it modify my repo without asking?**
`sdlc-docs` writes to `docs/sdlc/`. `ssdf-attest` writes to `docs/attestation/`. `ssdf-remediate` in default mode (`prompt-only`) writes only the prompt file; in `dispatch` mode it creates an isolated worktree. Nothing is force-pushed; nothing is committed to your working branch.

**Q: Can I run this in CI?**
Not directly — skills are interactive with Claude Code. But the *outputs* (`gaps.json`, attestation JSON) are designed to be CI-consumable: gate a PR on `posture_score >= 0.85`, fail the build if new criticals appear, etc.

**Q: Why three skills instead of one?**
Composability. You might want SDLC docs without an attestation, attestation without doc generation, or remediation against an externally-produced `gaps.json`. Single-responsibility skills compose; one mega-skill does not.

---

## Roadmap (suggested extensions)

- **SARIF output** for IDE/CI integration alongside the existing JSON.
- **CISA Common Form PDF** field-fill once the official template stabilises.
- **SLSA provenance level scoring** as a separate framework column.
- **Diff-mode attestation** — re-run shows posture delta vs previous run, not just current state.
- **Sub-skills for specific stacks** — e.g. `ssdf-attest-nodejs` with deeper npm-audit / lockfile analysis.

---

## Files in this package

```
skills/sdlc-docs/SKILL.md         # SDLC documentation generator
skills/ssdf-attest/SKILL.md       # Multi-framework attestation
skills/ssdf-remediate/SKILL.md    # Remediation prompt generator
README.md                         # This file
```

After install they live at `~/.claude/skills/<name>/SKILL.md`.

---

## Licence & disclaimer

Skills are local instruction files — use, fork, and adapt freely.

**Important:** Output from these skills is *advisory*. It is not a substitute for a signed CISA Secure Software Development Attestation Common Form, an IRAP assessment, an ISO 27001 audit, a SOC 2 report, or any other formal certification. Producing or relying on the output for compliance purposes is your responsibility, not Anthropic's and not the skill author's.
