# Setup Guide: Administrators

This guide is for the person rolling Claude Code out to an organization, usually IT, security, or an engineering lead. It walks through deploying the managed settings and CLAUDE.md files so the controls apply to every developer and can't be overridden.

The [README](README.md) explains why each control exists. Developers should follow [SETUP-DEVELOPERS.md](SETUP-DEVELOPERS.md).

## Before you start

You need:

- A **Claude Team or Enterprise** plan, if you want to lock logins to your organization. On individual Pro or Max plans, you can still enforce claude.ai logins, but not a specific organization.
- A way to push files or policy to developer machines: the claude.ai admin console, an MDM tool (Jamf, Intune, etc.), or the ability to place a system file.
- Your **Claude organization UUID**, from the claude.ai admin settings under Organization.
- A list of the **MCP servers** you approve, and the **package registries and internal hosts** your developers need to reach.
- One test machine per operating system you support.

## Step 1: Pick the right template

| Developer machines | Template |
|---|---|
| macOS, Linux, or Windows with WSL2 | `managed-settings-template.jsonc` |
| Native Windows (no WSL2) | `managed-settings-windows-template.jsonc` |

The main template runs every shell command in an OS-level sandbox and refuses to start where no sandbox is available, so it can't be used on native Windows. The Windows template drops the sandbox and adds stricter permission rules in its place. If you can, have Windows developers work inside WSL2 and use the main template.

## Step 2: Customize the template

Open the template and work through it top to bottom.

1. **Read the `_notes` block.** It explains what each setting does and when to change it.
2. **Replace every `YOUR_..._HERE` placeholder.** An unreplaced placeholder blocks access instead of allowing it.
   - `forceLoginOrgUUID`: your organization UUID. On Pro or Max plans, delete this key and keep `forceLoginMethod`.
   - `allowedMcpServers`: one entry per approved MCP server, matched by `serverUrl`.
3. **Add your hosts to `sandbox.network.allowedDomains`.** Include your package registries, artifact stores, and internal services. Anything missing gets blocked when developers run commands through Claude.
4. **Set `requiredMinimumVersion`** to the Claude Code version you've approved. Don't go below `2.0.65`, which includes the fixes for CVE-2025-59536 and CVE-2026-21852.
5. **Review the permission rules** (`deny` and `ask`) against how your teams actually work.
6. **Decide on auto mode.** The template turns it off (`permissions.disableAutoMode`). Remove that key only if you've evaluated auto mode and approved it.
7. **Add other approved plugins** to `enabledPlugins`. Project CodeGuard is already declared and force-enabled for everyone.
8. **Remove every key that starts with `_`** and all comments. The deployed file must be strict JSON.

Check that the result parses before you deploy it:

```bash
python3 -m json.tool managed-settings.json > /dev/null && echo OK
```

## Step 3: Deploy the settings

Use one of these methods.

| Method | Where it goes | Notes |
|---|---|---|
| **Server-managed settings** | claude.ai admin console | Easiest. The only method that also reaches Claude Code cloud sessions. Applies one policy to everyone, so it can't give native Windows a separate file. Settings arrive after first login, so also deploy `forceLoginMethod` and `forceLoginOrgUUID` by MDM or system file. |
| **MDM / OS policy** | macOS: plist domain `com.anthropic.claudecode`<br>Windows: registry `HKLM\SOFTWARE\Policies\ClaudeCode`, value `Settings` (the JSON) | Use this if you need different files for macOS and Windows. |
| **System file** | macOS: `/Library/Application Support/ClaudeCode/managed-settings.json`<br>Linux/WSL: `/etc/claude-code/managed-settings.json`<br>Windows: `C:\Program Files\ClaudeCode\managed-settings.json` | The old `C:\ProgramData\ClaudeCode\` path is no longer read. A `managed-settings.d/` folder next to the file can hold extra JSON files that merge in. |

## Step 4: Deploy the CLAUDE.md rules

1. Replace `[COMPANY NAME]` in the root `CLAUDE.md`.
2. Adjust the secrets-management tool references to match your stack, and the approved and prohibited licenses to match what Legal has approved.
3. Choose how it reaches developers:
   - **Managed memory (recommended):** place `CLAUDE.md` in the same system folder as `managed-settings.json`. It loads in every project, and users can't turn it off.
   - **Per repository:** commit it to the root of each repository.
4. Copy `mobile/CLAUDE.md` and `infra/CLAUDE.md` into the repositories that need them. They load only when Claude works in those folders.

## Step 5: Verify on a test machine

On each test machine, start Claude Code and check:

- [ ] `/status` shows the managed settings source.
- [ ] `claude doctor` reports no rejected entries.
- [ ] `/plugin` lists Project CodeGuard as installed.
- [ ] Logging in with a personal account or another organization is refused.
- [ ] A command that reaches a host outside `allowedDomains` (for example `curl https://example.com`) is blocked.
- [ ] On native Windows: Claude Code starts, and a PowerShell or `cmd /c` command prompts for approval.

## Step 6: Harden your repositories

Managed settings control what Claude Code can do on a laptop. Server-side controls catch whatever still gets through. For every repository:

- [ ] **Branch protection** on main and production branches: required reviews, required status checks, no bypassing.
- [ ] **CODEOWNERS** covering `CLAUDE.md`, `.claude/`, `.mcp.json`, `.github/workflows/`, Dockerfiles, and Terraform. These files can change how Claude behaves, so a security reviewer should approve changes to them.
- [ ] **CI security scanning**: secrets, SAST, and dependency audit. A starter workflow is in the [README](README.md#cicd-security-scanning), and more commands are in [`scripts/security-checks.md`](scripts/security-checks.md).

## Step 7: Tell your developers

Send developers [SETUP-DEVELOPERS.md](SETUP-DEVELOPERS.md) and let them know:

- Which account to log in with.
- Who to ask to add an MCP server, plugin, or network host.
- That `claude doctor` and `/status` are the first things to check when something is blocked.

## Keeping it current

- Raise `requiredMinimumVersion` when you approve a new Claude Code release.
- Review `allowedDomains`, `allowedMcpServers`, and `enabledPlugins` when teams ask for additions.
- After every change, check on a test machine again before rolling out widely.
