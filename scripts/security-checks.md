# Security Scripts and Commands

Quick-reference commands for security checks. Run these before committing code.

## Pre-Commit Secret Scanning

```bash
# Quick grep check for hardcoded secrets before committing
git diff --cached | grep -E "(password|secret|api_key|apikey|token)" -i
```

## Git Configuration Verification

```bash
# Verify .env is gitignored
grep -q ".env" .gitignore && echo "OK" || echo "ADD .env TO .gitignore"

# Verify company GitHub account
git config user.email  # Should be your company email
```

## License Scanning

```bash
# Check npm package licenses
npx license-checker-rseidelsohn --summary   # maintained fork of license-checker

# Check for prohibited licenses (matches the CLAUDE.md prohibited list; extend for -only/-or-later SPDX variants)
npx license-checker-rseidelsohn --failOn "GPL-2.0;GPL-3.0;AGPL-3.0;LGPL-2.1;LGPL-3.0;SSPL-1.0;BUSL-1.1;CC-BY-NC-4.0"

# Check Python package licenses
pip-licenses --format=markdown
```

## SBOM Generation

```bash
# Generate SBOM for Node.js project (CycloneDX format)
npx @cyclonedx/cyclonedx-npm --output-file sbom.json

# Generate SBOM for Python project
pip install cyclonedx-bom
cyclonedx-py environment --output sbom.json

# Generate SBOM for container images
syft <image> -o cyclonedx-json > sbom.json
```

## Container Security

```bash
# Scan container image for vulnerabilities
grype <image>

# Lint Dockerfile for best practices
hadolint Dockerfile

# Verify image signature (cosign)
cosign verify --key cosign.pub <image>
```

## Infrastructure Security

```bash
# Scan Terraform for misconfigurations (tfsec is deprecated; its checks now live in Trivy)
# Trivy had a supply-chain compromise in March 2026 (malicious v0.69.4): pin a verified release
trivy config .

# Scan Kubernetes manifests
kubesec scan deployment.yaml

# Validate CloudFormation templates
cfn-lint template.yaml
```

## SAST (Static Application Security Testing)

```bash
# Run Semgrep with OWASP rules
semgrep --config=p/owasp-top-ten .

# Run CodeQL (via GitHub Actions - add to .github/workflows/)
# See: https://docs.github.com/en/code-security/code-scanning/creating-an-advanced-setup-for-code-scanning

# Run Bandit for Python
bandit -r . -f json -o bandit-report.json

# Run ESLint security plugin for JavaScript/TypeScript
npx eslint --plugin security .
```

## DAST (Dynamic Application Security Testing)

```bash
# Run OWASP ZAP baseline scan against staging
docker run -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t https://staging.example.com

# Run Nuclei with common vulnerability templates
nuclei -u https://staging.example.com -tags cve,vuln
```

## Secret Scanning (Pre-Commit)

```bash
# Install and run Gitleaks
gitleaks detect --source . --report-path gitleaks-report.json

# Install and run TruffleHog
trufflehog filesystem . --json > trufflehog-report.json

# Run git-secrets (AWS-focused)
git secrets --scan
```

## GitHub Actions Security

```bash
# Audit GitHub Actions for non-SHA-pinned versions
# This finds any "uses:" lines that reference a tag (e.g., @v4) instead of a commit SHA
grep -rn "uses:.*@" .github/workflows/*.yml | grep -vE "@[a-f0-9]{40}"
# Any results here are using tag-based references instead of SHA pins
```

## Dependency Audit

```bash
# Audit npm dependencies for vulnerabilities
npm audit --json > npm-audit-report.json

# Audit Python dependencies
pip-audit --format json --output pip-audit-report.json

# Audit Go dependencies
govulncheck ./...

# Audit Rust dependencies
cargo audit
```

## Claude Code Configuration Checks

```bash
# Confirm which settings sources are loaded and whether a managed policy applies
claude doctor        # lists rejected settings entries and config errors
# Inside a session: /status shows "Setting sources"; /permissions lists effective rules

# Before trusting an unfamiliar repository, inspect the files that can run code or redirect traffic
ls -la .claude/ 2>/dev/null; cat .mcp.json .claude/settings.json 2>/dev/null
grep -rn "ANTHROPIC_BASE_URL\|apiKeyHelper\|\"hooks\"" .claude/ .mcp.json 2>/dev/null
```
