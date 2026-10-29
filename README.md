# Claude Code Enterprise Security Framework

Enterprise governance framework for securing Claude Code deployments. Provides layered security controls that wrap around [Project CodeGuard](https://project-codeguard.org/) technical rules and align with OWASP GenAI security guidance.

## What This Is

A set of CLAUDE.md files and supporting documents that enforce security policies when teams use Claude Code for AI-assisted development. The framework operates at three layers:

1. **Platform enforcement** (`managed-settings-template.jsonc`): Controls what Claude Code CAN DO (OS-level sandbox, tool permissions, file access restrictions, approved accounts, MCP servers and plugins). Deployed via server-managed settings or MDM. Users cannot override.
2. **Technical code security** (Project CodeGuard plugin): Controls HOW code is generated (crypto, input validation, auth patterns, API security, container hardening). Open-source rules from CoSAI/OWASP.
3. **Enterprise governance** (CLAUDE.md files): Controls organizational policy (company accounts, secrets management tools, license compliance, AI agent risk controls).

## File Structure

```
├── CLAUDE.md                          # Root: Universal rules (loads every interaction)
├── mobile/CLAUDE.md                   # Mobile platform rules (loads in mobile directories)
├── infra/CLAUDE.md                    # Cloud, container, K8s, CI/CD rules (loads in infra directories)
├── docs/security-patterns/
│   ├── secrets-management.md          # WRONG/CORRECT code examples (loaded on demand)
│   └── agentic-security.md            # OWASP LLM Top 10 + Agentic Top 10 mapping
├── scripts/security-checks.md         # Pre-commit, SBOM, scanning commands
└── managed-settings-template.jsonc    # Platform-level permission enforcement template
```

## Quick Start (Managed / Enterprise)

For organizations on a Claude Team or Enterprise plan, or with MDM (Jamf, Intune, etc.):

1. Clone this repo
2. Customize `[COMPANY NAME]` placeholder in `CLAUDE.md`
3. Install Project CodeGuard: `/plugin marketplace add cosai-oasis/project-codeguard`
4. Customize `managed-settings-template.jsonc`. Replace every `YOUR_..._HERE` placeholder (org UUID, approved MCP servers), add your package registries to the sandbox `allowedDomains`, and read the `_notes` block. Then remove every key starting with `_`, because managed settings must be strict JSON.
5. Deploy it through one of:
   - **Server-managed settings** in the claude.ai admin console. This is the only managed source that also reaches Claude Code cloud sessions.
   - **MDM / OS policy**: macOS plist domain `com.anthropic.claudecode`; Windows registry `HKLM\SOFTWARE\Policies\ClaudeCode` (value `Settings`, holding the JSON).
   - **System file**:
     - macOS: `/Library/Application Support/ClaudeCode/managed-settings.json`
     - Linux/WSL: `/etc/claude-code/managed-settings.json`
     - Windows: `C:\Program Files\ClaudeCode\managed-settings.json` (the old `C:\ProgramData\ClaudeCode\` path is no longer read)
     - A `managed-settings.d/` directory next to the file can hold additional drop-in JSON files that merge with it.
6. Optionally deploy the root `CLAUDE.md` as **managed memory** so it loads in every project without being copied: place it as `CLAUDE.md` in the same system directory as `managed-settings.json`. Users cannot exclude managed memory.
7. Copy the `mobile/` and `infra/` CLAUDE.md files into the project repositories that need them
8. Verify on a test machine: `/status` shows the managed source, and `claude doctor` lists any rejected entries.

Managed settings cannot be overridden by users. This is the recommended approach for enterprises.

**Native Windows:** the template sets `sandbox.failIfUnavailable: true`, and the sandbox runs only on macOS, Linux, and WSL2. Claude Code will refuse to start on native Windows with this template. Either standardize Windows developers on WSL2, or deploy a Windows-specific copy with `failIfUnavailable: false` and accept that Bash commands run unsandboxed there.

## Quick Start (Non-Managed / Individual / Small Team)

For individual developers, small teams, or environments without MDM:

### 1. Copy CLAUDE.md files into your project

```bash
# From the root of your project repository
cp -r /path/to/Claude-Security-Instructions/CLAUDE.md .
cp -r /path/to/Claude-Security-Instructions/mobile/ ./mobile/       # if building mobile apps
cp -r /path/to/Claude-Security-Instructions/infra/ ./infra/         # if writing infrastructure code
cp -r /path/to/Claude-Security-Instructions/docs/ ./docs/           # security patterns reference
```

Customize `[COMPANY NAME]` to your team or project name. Remove sections that don't apply (e.g., "Company Device and Account Requirements" for personal projects).

### 2. Install Project CodeGuard

In Claude Code, run:

```
/plugin marketplace add cosai-oasis/project-codeguard
/plugin install codeguard-security@project-codeguard
```

### 3. Configure permission rules

Without MDM, use **project-level** or **user-level** settings files to enforce permission rules.

**Project-level** (`.claude/settings.json` — commit to git, shared with collaborators):

```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(sudo:*)",
      "Bash(git push --force:*)",
      "Read(**/.env)",
      "Read(**/*.pem)"
    ],
    "ask": [
      "Bash(curl:*)",
      "Bash(git push:*)"
    ]
  }
}
```

The example above is a starter set. For the full enterprise rule list — network exfiltration tooling (`nc`, `ncat`, `telnet`, `nslookup`, `dig`, `base64`), additional secret-file globs, container/IaC `ask` rules — copy from `managed-settings-template.jsonc` and adapt. The same `permissions` and `sandbox` schema works in `.claude/settings.json`. Managed-only keys (`allowManagedHooksOnly`, `allowManagedMcpServersOnly`, `strictKnownMarketplaces`, `requiredMinimumVersion`) are ignored in project and user files.

**Turn on the sandbox too.** Bash deny rules match command text, so `Bash(rm -rf:*)` does not stop `/bin/rm -rf`, and Read deny rules don't stop indirect reads like `grep -r` across the project. The sandbox enforces filesystem and network limits at the OS level for every command Claude runs (macOS, Linux, WSL2):

```json
{
  "sandbox": {
    "enabled": true,
    "network": {
      "allowedDomains": ["github.com", "registry.npmjs.org"]
    }
  }
}
```

**User-level** (`~/.claude/settings.json` — applies to all your projects, not shared):

```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf:*)",
      "Bash(sudo:*)",
      "Read(**/.env)",
      "Read(**/*.pem)",
      "Read(**/*.key)"
    ]
  }
}
```

### Settings precedence (highest to lowest)

| Priority | File | Scope | Shared via git? |
|---|---|---|---|
| 1 | Managed (server-managed, MDM policy, or `managed-settings.json`) | All users it is deployed to | No |
| 2 | `--settings` on the command line | This session only | No |
| 3 | `.claude/settings.local.json` | This project, this user only | No (gitignored) |
| 4 | `.claude/settings.json` | This project, all collaborators | Yes |
| 5 | `~/.claude/settings.json` | All projects for this user | No |

Permission rule lists merge across all levels rather than overriding each other. Rules are evaluated in order: **deny > ask > allow**. In a shared `.claude/settings.json`, `deny` and `ask` rules apply immediately, but `allow` rules and most `env` values wait until each person trusts the folder.

### Key differences from managed deployment

| Capability | Managed (MDM) | Non-Managed |
|---|---|---|
| Permission rules (deny/ask/allow) | Yes | Yes |
| Sandbox (`sandbox.enabled`) | Yes, lockable (`failIfUnavailable`, `allowUnsandboxedCommands: false`, `allowManagedDomainsOnly`) | Yes, but users can turn it off |
| Users cannot override rules | Yes | No — users can modify their own settings |
| `disableBypassPermissionsMode` / `disableAutoMode` | Yes, enforced | Yes, but users can remove it |
| `forceLoginMethod` / `forceLoginOrgUUID` (company accounts only) | Yes | Not enforced |
| `allowManagedPermissionRulesOnly`, `allowManagedHooksOnly`, `allowManagedMcpServersOnly` | Yes | Not available (managed-only) |
| `strictKnownMarketplaces`, `requiredMinimumVersion` | Yes | Not available (managed-only) |
| CLAUDE.md governance rules | Yes | Yes |
| Project CodeGuard plugin | Yes | Yes |

In non-managed environments, permission rules are **advisory** — a developer can modify their local settings. Security enforcement relies on:

- **CLAUDE.md rules** (Claude Code follows these as instructions during code generation)
- **Project-level `.claude/settings.json`** (shared with the team via git)
- **Code review** (catches anything that slips through)
- **CI/CD security scanning** (SAST, SCA, secret scanning — see `scripts/security-checks.md`)

### 4. Harden CI/CD and repository settings (strongly recommended)

Without managed-settings.json, CI/CD and repository controls become your **primary hard enforcement layer** — the only controls a developer or prompt injection cannot bypass locally.

#### Branch protection rules

Configure on your main/production branches (GitHub Settings > Branches > Branch protection):

- Require pull request reviews before merging (at least 1 reviewer)
- Require status checks to pass (wire up the CI jobs below)
- Do not allow bypassing the above settings
- Require signed commits (optional but recommended)

#### CODEOWNERS for security-sensitive files

Create `.github/CODEOWNERS` to require specific reviewers for governance files:

```
# Security-sensitive files require team lead / security review
CLAUDE.md                     @your-team-lead
**/CLAUDE.md                  @your-team-lead
.claude/                      @your-team-lead
.mcp.json                     @your-team-lead
.github/workflows/            @your-team-lead
.github/CODEOWNERS            @your-team-lead
Dockerfile                    @your-team-lead
*.tf                          @your-team-lead
```

#### CI/CD security scanning

Add these jobs to `.github/workflows/security.yml` (or equivalent). These are the **hard controls** that catch issues regardless of local settings:

```yaml
name: Security checks
on: [pull_request]

permissions:
  contents: read

# Pin every action by full commit SHA in production (see infra/CLAUDE.md).
jobs:
  secret-scanning:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
      - name: Scan for secrets
        # gitleaks-action is under a commercial EULA (free key from gitleaks.io for organizations).
        # If Legal won't approve that, run the MIT-licensed gitleaks binary from its GitHub release instead.
        uses: gitleaks/gitleaks-action@v3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}  # organizations only

  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Run Semgrep OWASP rules
        run: |
          pip install semgrep
          semgrep scan --config=p/owasp-top-ten --error .

  dependency-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Audit dependencies
        run: npm audit --audit-level=high  # adjust for your package manager

  governance-file-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
      - name: Flag governance file changes
        run: |
          # Warn if security-sensitive files were modified
          CHANGED=$(git diff --name-only origin/main...HEAD | grep -E '(CLAUDE\.md|\.claude/|\.mcp\.json|managed-settings|managed-mcp|\.github/workflows/|\.github/CODEOWNERS|\.gitignore|Dockerfile|\.tf$)' || true)
          if [ -n "$CHANGED" ]; then
            echo "::warning::Security-sensitive files modified — require security review:"
            echo "$CHANGED"
          fi
```

#### Pre-commit hooks (local enforcement)

For developers who want local checks before commits reach CI:

```bash
# Install pre-commit framework
pip install pre-commit

# Create .pre-commit-config.yaml
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0   # pin to a specific version
    hooks:
      - id: gitleaks
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: detect-private-key
      - id: check-added-large-files
EOF

pre-commit autoupdate   # bumps the revs above to the latest releases
pre-commit install
```

#### Summary: non-managed defense layers

| Layer | Enforcement | Can developer bypass? | Catches prompt injection? |
|---|---|---|---|
| `.claude/settings.json` deny rules | Claude Code runtime | Yes (can edit local settings) | Partially — blocks tool use but developer can override |
| CLAUDE.md instructions | Model behavior | N/A (model may not follow if injected) | Weakly — soft defense |
| Pre-commit hooks | Local git hooks | Yes (`--no-verify`) | Yes for secrets, not for logic |
| **CI/CD scanning (SAST, secrets, deps)** | **GitHub/CI platform** | **No (runs server-side)** | **Yes — catches committed issues** |
| **Branch protection + required reviews** | **GitHub/CI platform** | **No (enforced by platform)** | **Yes — human catches what automation misses** |
| **CODEOWNERS** | **GitHub platform** | **No (enforced by platform)** | **Yes — protects governance files** |

The bolded rows are the **hard controls** in a non-managed environment. CI/CD scanning, branch protection, and CODEOWNERS are enforced server-side and cannot be bypassed by a local prompt injection or a compromised developer workstation.

## Standards Alignment

- [Project CodeGuard](https://project-codeguard.org/) (CoSAI/OWASP) for technical code security
- [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) for AI application risks (mapping uses 2025 IDs; 2026 edition published August 2026)
- [OWASP Top 10 for Agentic Applications (2026)](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/) for autonomous AI risks
- [Claude Code security](https://code.claude.com/docs/en/security), [permissions](https://code.claude.com/docs/en/permissions), [sandboxing](https://code.claude.com/docs/en/sandboxing), and [managed settings](https://code.claude.com/docs/en/managed-settings) docs
- [OWASP GenAI Security Project](https://genai.owasp.org/) for broader governance guidance

## Customization

Before deploying to a client or organization:

- Replace `[COMPANY NAME]` in root CLAUDE.md
- Adjust secrets management tool references to match the client's stack
- Review and customize managed-settings.json deny/ask rules for the org's workflows
- Set the org UUID, approved MCP servers, sandbox `allowedDomains`, and `requiredMinimumVersion`
- Decide whether to allow `auto` permission mode (the template disables it)
- Add or remove approved licenses based on Legal team input
- Update MCP server governance to reflect approved integrations

## License

Apache 2.0
