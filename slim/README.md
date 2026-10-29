# Claude Code Security Framework (Slim)

Security governance framework for Claude Code deployments. This is the slim version — core security essentials for web and API development. For the full enterprise version (mobile, infrastructure, compliance mapping, incident response), see the parent directory.

## What This Is

CLAUDE.md files and supporting documents that enforce security policies when teams use Claude Code. The framework operates at two layers:

1. **Platform enforcement** (`managed-settings-template.jsonc`): Controls what Claude Code CAN DO (OS-level sandbox, tool permissions, file access restrictions, approved accounts, MCP servers and plugins). Deployed via server-managed settings or MDM. Users cannot override.
2. **Enterprise governance** (CLAUDE.md): Controls organizational policy (company accounts, secrets management, license compliance, AI agent risk controls, web/API security).

Both layers work alongside [Project CodeGuard](https://project-codeguard.org/) for technical code security patterns.

## File Structure

```
├── CLAUDE.md                          # Security rules (loads every interaction)
├── managed-settings-template.jsonc    # Platform enforcement template (macOS, Linux, WSL2)
├── managed-settings-windows-template.jsonc  # Native Windows variant (no OS sandbox)
├── .gitignore                         # Prevents secrets from being committed
├── README.md                          # This file
└── docs/security-patterns/
    └── secrets-management.md          # WRONG/CORRECT code examples
```

## Quick Start (Managed / Enterprise)

For organizations with MDM (Jamf, Intune, etc.):

1. Clone this repo
2. Customize `[COMPANY NAME]` in `CLAUDE.md`
3. Install Project CodeGuard: `/plugin marketplace add cosai-oasis/project-codeguard`
4. Customize `managed-settings-template.jsonc` (replace every `YOUR_..._HERE` placeholder, add your registries to the sandbox `allowedDomains`, read `_notes`), remove every key starting with `_`, then deploy it via server-managed settings (claude.ai admin console), MDM, or the system file:
   - macOS: `/Library/Application Support/ClaudeCode/managed-settings.json`
   - Linux: `/etc/claude-code/managed-settings.json`
   - Windows: `C:\Program Files\ClaudeCode\managed-settings.json` (native Windows has no OS sandbox: deploy `managed-settings-windows-template.jsonc` there instead)
5. Copy `CLAUDE.md` into your project repositories, or deploy it as managed memory (`CLAUDE.md` in the same system directory)

## Quick Start (Non-Managed / Small Team)

1. Copy `CLAUDE.md` into the root of your project repo
2. Customize `[COMPANY NAME]` to your team or project name
3. Install Project CodeGuard in Claude Code
4. Configure `.claude/settings.json` in your project for permission rules:

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

The example above is a starter set. For the full slim rule list (network exfiltration tooling, additional secret-file globs, container/IaC `ask` rules), copy from `slim/managed-settings-template.jsonc` and adapt.

Bash deny rules match command text (`Bash(rm -rf:*)` does not stop `/bin/rm -rf`), so also enable the sandbox in the same file: `"sandbox": {"enabled": true, "network": {"allowedDomains": ["github.com", "registry.npmjs.org"]}}` (macOS, Linux, WSL2).

### Settings Precedence (highest to lowest)

| Priority | File | Scope |
|---|---|---|
| 1 | Managed (server-managed, MDM policy, or `managed-settings.json`) | All users it is deployed to |
| 2 | `--settings` on the command line | This session only |
| 3 | `.claude/settings.local.json` | This project, this user only |
| 4 | `.claude/settings.json` | This project, all collaborators |
| 5 | `~/.claude/settings.json` | All projects for this user |

Permission rule lists merge across levels. Rules are evaluated in order: **deny > ask > allow**.

## Customization

Before deploying:

- Replace `[COMPANY NAME]` in CLAUDE.md
- Adjust secrets management tool references to match your cloud provider
- Review and customize managed-settings deny/ask rules for your workflows
- Add or remove approved licenses based on Legal team input

## License

Apache 2.0
