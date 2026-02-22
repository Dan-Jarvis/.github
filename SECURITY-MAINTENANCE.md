# Security Pipeline Maintenance Guide

Maintenance reminders are automatically created as GitHub issues:
- **Quarterly** — via `reminder-quarterly.yml`

Dependabot automatically opens weekly PRs to keep GitHub Actions up to date.
Review and merge these PRs as they arrive — your security pipeline runs on each one automatically.

---

## Ongoing Tasks (Triggered by Dependabot)

Dependabot opens PRs every Monday for GitHub Actions SHA pin updates. For each one:
- Check the security pipeline passed on the PR
- Merge it

If no Dependabot PRs have arrived in over a week, manually check:
- https://github.com/gitleaks/gitleaks-action/commits/main
- https://github.com/aquasecurity/trivy-action/commits/main

---

## Quarterly Tasks

### 1. Review Harden Runner Egress Logs
- Check the StepSecurity dashboard for unexpected outbound calls
- If a job's egress is stable, consider switching from `egress-policy: audit` to `egress-policy: block` with explicit `allowed-endpoints`
- Prioritise the `secrets` job (handles `GITHUB_TOKEN`)

### 2. Check Action Version Bumps
Dependabot handles patch updates but major version bumps (e.g. v4 → v5) need manual review:

| Action | Current | Check |
|--------|---------|-------|
| actions/checkout | v4 | https://github.com/actions/checkout/releases |
| actions/upload-artifact | v4 | https://github.com/actions/upload-artifact/releases |
| actions/download-artifact | v4 | https://github.com/actions/download-artifact/releases |
| actions/dependency-review-action | v4 | https://github.com/actions/dependency-review-action/releases |
| github/codeql-action | v3 | https://github.com/github/codeql-action/releases |
| step-security/harden-runner | v2 | https://github.com/step-security/harden-runner/releases |
| ossf/scorecard-action | v2 | https://github.com/ossf/scorecard-action/releases |
| gitleaks/gitleaks-action | v2 | https://github.com/gitleaks/gitleaks-action/releases |
| aquasecurity/trivy-action | 0.x | https://github.com/aquasecurity/trivy-action/releases |

### 3. Check Anthropic Model Deprecations
- Check https://docs.anthropic.com/en/docs/resources/model-deprecations
- If a model is deprecated, update the `model` input in the affected repo's `.github/workflows/security.yml`

### 4. Rotate Anthropic API Key
1. Go to https://console.anthropic.com and create a new API key named `github-security-scanner`
2. Go to GitHub Organisation Settings → Secrets → Actions
3. Update `ANTHROPIC_API_KEY` with the new key
4. Delete the old key from Anthropic Console
5. Verify the next PR scan completes successfully

### 5. Review False Positives
- Check recent security scan PR comments across your repos
- If a finding is a confirmed false positive:
  - **Trivy**: add the CVE to `.trivyignore` in the affected repo
  - **Gitleaks**: add a rule to `.gitleaks.toml` in the affected repo

### 6. Sync Scorecard Workflow Across Repos
`scorecard.yml` must be copied into each repo (it can't be a reusable workflow). When you update the `.github` repo's copy, propagate changes to all consumer repos.

### 7. Check for New Repos
- Review your repos and confirm any new ones have the caller workflow added
- See **Adding the Pipeline to a New Repo** below

---

## Adding the Pipeline to a New Repo

Create `.github/workflows/security.yml` in the new repo:
```yaml
name: Security Scanning

on:
  pull_request:
    branches: [main]

jobs:
  security:
    uses: Dan-Jarvis/.github/.github/workflows/reusable-security.yml@main
    with:
      language: python  # adjust — see Recommended Scanner Inputs below
      model: claude-haiku-4-5-20251001
      repo_context: 'Brief description of what this repo does'
      run_codeql: true
      run_secrets: true
      run_dependencies: true
      run_container: false  # enable only if repo uses containers
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Recommended Scanner Inputs by Repo Type

| Repo type | `language` | `run_codeql` | `run_secrets` | `run_dependencies` | `run_container` |
|-----------|-----------|--------------|---------------|--------------------|-----------------|
| WordPress (PHP) | `php` | true | true | true | false |
| Node.js app | `javascript-typescript` | true | true | true | true |
| Python app | `python` | true | true | true | true |
| Static site / docs | — | false | true | false | false |

> **Note:** `run_dependencies` only works on `pull_request` events. It is automatically skipped on other triggers.

### Adding OSSF Scorecard to a Repo

Copy `.github/workflows/scorecard.yml` from this repo into the target repo at the same path. This runs on push to main and weekly on Saturday — it cannot be a reusable `workflow_call` workflow because it needs `schedule` and `push` triggers to publish results and generate badges.

The workflow uploads SARIF results to GitHub's code-scanning dashboard automatically.

### Supported CodeQL Languages

| Language | Input value |
|----------|-------------|
| Python | `python` |
| JavaScript / TypeScript | `javascript-typescript` |
| PHP | `php` |
| Go | `go` |
| Ruby | `ruby` |
| Java | `java` |

---

## Troubleshooting

**Claude API returns 401** — `ANTHROPIC_API_KEY` secret is missing or incorrect. Check Organisation secrets.

**Claude API returns 429** — Rate limit hit. Scan summary will still be posted, Claude analysis skipped.

**CodeQL autobuild fails** — Add a manual build step between `init` and `analyze` in the caller workflow.

**Gitleaks flags a test file** — Add a `.gitleaks.toml` to the repo root with an allowlist for that path.

**Trivy flags an accepted vulnerability** — Add the CVE to a `.trivyignore` file in the repo root.

**Dependency review fails on push events** — This is expected. `dependency-review-action` only works on `pull_request` triggers and is automatically skipped otherwise.
