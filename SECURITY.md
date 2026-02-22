# Security Policy

## Scope

This repository contains org-level GitHub Actions workflows that run across all `Dan-Jarvis` repositories. A vulnerability here could affect the CI/CD pipeline of every consumer repo.

## Supported Versions

Only the `main` branch is supported. There are no versioned releases.

## Reporting a Vulnerability

**Do not open a public issue.**

Please use [GitHub Security Advisories](https://github.com/Dan-Jarvis/.github/security/advisories/new) to report vulnerabilities privately.

You can expect:
- **Acknowledgement** within 72 hours
- **Resolution or mitigation plan** within 14 days
- **Credit** in the fix commit (unless you prefer anonymity)

## What Qualifies

- Workflow injection vulnerabilities (shell injection via `${{ }}` expressions)
- Secrets exposure or exfiltration paths
- Permission escalation in reusable workflows
- Supply-chain risks (unpinned actions, compromised dependencies)

## What Doesn't Qualify

- Findings already tracked in `SECURITY-MAINTENANCE.md`
- Informational scanner output (false positives, low-severity CVEs in dev dependencies)
