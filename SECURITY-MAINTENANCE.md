# Security Pipeline Maintenance Guide

This document describes the maintenance tasks required to keep the security scanning pipeline healthy across all repositories.

Maintenance reminders are automatically created as GitHub issues on a schedule:
- **Monthly** — via `reminder-monthly.yml`
- **Quarterly** — via `reminder-quarterly.yml`

---

## Monthly Tasks

### 1. Update Action SHA Pins
Two third-party actions are pinned to specific commit SHAs to protect against supply chain attacks. Check monthly for updates:

**gitleaks/gitleaks-action**
1. Go to https://github.com/gitleaks/gitleaks-action/commits/main
2. Copy the latest commit SHA (the full 40-character hash)
3. Update in `.github/workflows/reusable-security.yml`:
```yaml
   uses: gitleaks/gitleaks-action@YOUR_NEW_SHA
```

**aquasecurity/trivy-action**
1. Go to https://github.com/aquasecurity/trivy-action/commits/main
2. Copy the latest commit SHA
3. Update in `.github/workflows/reusable-security.yml`:
```yaml
   uses: aquasecurity/trivy-action@YOUR_NEW_SHA
```

### 2. Check Anthropic Model Deprecations
1. Check https://docs.anthropic.com/en/docs/resources/model-deprecations
2. If any model used in a caller workflow is deprecated, update the `model` input in that repo's `.github/workflows/security.yml`

### 3. Review False Positives
- Check recent PR comments from the security pipeline across your repos
- If a finding is a confirmed false positive:
  - **Trivy**: add the CVE to `.trivyignore` in the affected repo
  - **Gitleaks**: add a rule to `.gitleaks.toml` in the affected repo

---

## Quarterly Tasks

### 1. Update Action Version Tags
Check for major version bumps on GitHub-maintained actions:

| Action | Current | Check |
|--------|---------|-------|
| actions/checkout | v4 | https://github.com/actions/checkout/releases |
| actions/upload-artifact | v4 | https://github.com/actions/upload-artifact/releases |
| actions/download-artifact | v4 | https://github.com/actions/download-artifact/releases |
| actions/dependency-review-action | v4 | https://github.com/actions/dependency-review-action/releases |
| github/codeql-action | v3 | https://github.com/github/codeql-action/releases |

Update the version tag in `.github/workflows/reusable-security.yml` if a new major version is available.

### 2. Rotate Anthropic API Key
1. Go to https://console.anthropic.com and create a new API key named `github-security-scanner`
2. Go to GitHub Organisation Settings → Secrets → Actions
3. Update `ANTHROPIC_API_KEY` with the new key
4. Return to Anthropic Console and delete the old key
5. Verify the next PR scan completes successfully

### 3. Review False Positives Across All Repos
- Do a broader review of `.trivyignore` and `.gitleaks.toml` files across all repos
- Remove any suppressions that are no longer valid

### 4. Check for New Repos
- Review your GitHub repos and confirm any new repos have the caller workflow added
- Copy `.github/workflows/security.yml` from an existing repo and adjust `language`, `model`, and `repo_context` inputs

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
      model: claude-haiku-4-5-20251001  # haiku for cost, sonnet for better analysis
      repo_context: 'Brief description of what this repo does'
      run_codeql: true
      run_secrets: true
      run_dependencies: true
      run_container: true
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

---

## Supported CodeQL Languages

| Language | Input value |
|----------|-------------|
| Python | `python` |
| JavaScript / TypeScript | `javascript-typescript` |
| PHP | `php` |
| Go | `go` |
| Ruby | `ruby` |
| Java | `java` |
| C / C++ | `cpp` |

---

## Troubleshooting

**Claude API returns 401** — `ANTHROPIC_API_KEY` secret is missing or incorrect. Check Organisation secrets.

**Claude API returns 429** — Rate limit hit due to multiple concurrent PRs. The scan summary will still be posted, Claude analysis will be skipped.

**CodeQL autobuild fails** — The repo may need a manual build step. Add a custom build step between `init` and `analyze` in the reusable workflow or the caller workflow.

**Gitleaks flags a test file** — Add a `.gitleaks.toml` to the repo root with an allowlist rule for that file path.

**Trivy flags an accepted vulnerability** — Add the CVE ID to a `.trivyignore` file in the repo root, one CVE per line.
