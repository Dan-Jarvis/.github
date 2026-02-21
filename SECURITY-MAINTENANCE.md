# Security Pipeline Maintenance Guide

Maintenance reminders are automatically created as GitHub issues:
- **Monthly** — via `reminder-monthly.yml`
- **Quarterly** — via `reminder-quarterly.yml`

Dependabot automatically opens weekly PRs to keep GitHub Actions up to date.
Review and merge these PRs as they arrive — your security pipeline runs on each one automatically.

---

## Monthly Tasks

### 1. Review and Merge Dependabot PRs
Dependabot opens PRs automatically every Monday. For each one:
- Check the security pipeline passed on the PR
- Merge it

If no Dependabot PRs have arrived in over a week, manually check:
- https://github.com/gitleaks/gitleaks-action/commits/main
- https://github.com/aquasecurity/trivy-action/commits/main

### 2. Check Anthropic Model Deprecations
- Check https://docs.anthropic.com/en/docs/resources/model-deprecations
- If a model is deprecated, update the `model` input in the affected repo's `.github/workflows/security.yml`

### 3. Review False Positives
- Check recent security scan PR comments across your repos
- If a finding is a confirmed false positive:
  - **Trivy**: add the CVE to `.trivyignore` in the affected repo
  - **Gitleaks**: add a rule to `.gitleaks.toml` in the affected repo

---

## Quarterly Tasks

### 1. Check Action Version Bumps
Dependabot handles patch updates but major version bumps (e.g. v4 → v5) need manual review:

| Action | Current | Check |
|--------|---------|-------|
| actions/checkout | v4 | https://github.com/actions/checkout/releases |
| actions/upload-artifact | v4 | https://github.com/actions/upload-artifact/releases |
| actions/download-artifact | v4 | https://github.com/actions/download-artifact/releases |
| actions/dependency-review-action | v4 | https://github.com/actions/dependency-review-action/releases |
| github/codeql-action | v3 | https://github.com/github/codeql-action/releases |

### 2. Rotate Anthropic API Key
1. Go to https://console.anthropic.com and create a new API key named `github-security-scanner`
2. Go to GitHub Organisation Settings → Secrets → Actions
3. Update `ANTHROPIC_API_KEY` with the new key
4. Delete the old key from Anthropic Console
5. Verify the next PR scan completes successfully

### 3. Check for New Repos
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
    uses: YOUR_GITHUB_USERNAME/.github/.github/workflows/reusable-security.yml@main
    with:
      language: python  # adjust to repo language
      model: claude-haiku-4-5-20251001
      repo_context: 'Brief description of what this repo does'
      run_codeql: true
      run_secrets: true
      run_dependencies: true
      run_container: true
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

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
