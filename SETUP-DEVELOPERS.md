# Setup Guide: Developers

This guide is for developers using Claude Code under this security framework. Find your situation below and follow that section.

- **Your organization manages Claude Code for you:** see [Managed setup](#managed-setup). It takes a few minutes.
- **No one manages it** (individual developers or small teams): see [Self-managed setup](#self-managed-setup).

Not sure which? Run `/status` in Claude Code. If it lists a managed settings source, your organization manages it.

Administrators should follow [SETUP-ADMINS.md](SETUP-ADMINS.md) instead.

---

## Managed setup

Your administrator has already applied the security settings and the CodeGuard plugin. You can't change these settings, and that's intentional.

### 1. Use your company accounts

- Work on your company-issued computer.
- Log in to Claude Code with your **company** claude.ai account. Personal accounts and other organizations are blocked.
- Check that Git commits use your company email:

  ```bash
  git config user.email
  ```

### 2. Check your setup

In Claude Code, run:

- `/status`: should show the managed settings source.
- `/plugin`: should list Project CodeGuard.

In a terminal, run:

```bash
claude doctor
```

It should report no problems. Send any errors to your administrator.

### 3. Know what to expect

- **Some commands prompt for approval**, such as `git push`, `curl`, and container or infrastructure commands. Read what Claude wants to run before you approve.
- **Some commands are blocked**, such as `sudo`, `rm -rf`, and force-pushing. So is reading secret files like `.env` and SSH keys.
- **Network access is limited** to approved hosts when Claude runs commands. If an install or build fails because a host is blocked, ask your administrator to add it. Don't try to work around it.
- **MCP servers and plugins** must be approved centrally. Ask your administrator rather than adding your own.

Skip to [Rules to follow either way](#rules-to-follow-either-way).

---

## Self-managed setup

Without managed settings, these controls are guardrails you set up yourself. You (or anyone else) can still change them later. Your repository's CI checks and branch protection are what actually enforce the rules.

### 1. Add the CLAUDE.md files to your project

From your project's root folder:

```bash
cp /path/to/VCG-Claude-Security-Instructions/CLAUDE.md .
```

```bash
cp -r /path/to/VCG-Claude-Security-Instructions/docs/ ./docs/
```

If you build mobile apps or write infrastructure code, also copy `mobile/CLAUDE.md` or `infra/CLAUDE.md` into the matching folders.

Then open `CLAUDE.md`, replace `[COMPANY NAME]`, and remove sections that don't apply to you.

### 2. Install Project CodeGuard

In Claude Code, run:

```
/plugin marketplace add cosai-oasis/project-codeguard
```

```
/plugin install codeguard-security@project-codeguard
```

### 3. Add permission rules and turn on the sandbox

Create `.claude/settings.json` in your project and commit it, so everyone on the project gets the same rules:

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
  },
  "sandbox": {
    "enabled": true,
    "network": {
      "allowedDomains": ["github.com", "registry.npmjs.org"]
    }
  }
}
```

Add the package registries you use to `allowedDomains`. The sandbox works on macOS, Linux, and WSL2.

**Why the sandbox matters:** deny rules only match the command text, so `Bash(rm -rf:*)` doesn't stop `/bin/rm -rf`. The sandbox enforces file and network limits at the OS level, whatever command is run.

For a fuller rule set, copy the rules you need from `managed-settings-template.jsonc`. The `permissions` and `sandbox` sections work the same way in `.claude/settings.json`.

### 4. Protect your repository

These controls run on GitHub, so a misconfigured laptop or a prompt injection can't bypass them.

- [ ] **Branch protection** on your main branch: require pull request reviews and passing status checks.
- [ ] **CODEOWNERS** (`.github/CODEOWNERS`) so changes to `CLAUDE.md`, `.claude/`, `.mcp.json`, and `.github/workflows/` need a reviewer's approval.
- [ ] **CI security scanning** for secrets, code, and dependencies. Copy the starter workflow from the [README](README.md#cicd-security-scanning).
- [ ] **Pre-commit hooks** (optional) to catch secrets before you commit. Setup steps are in the [README](README.md#pre-commit-hooks-local-enforcement).

---

## Rules to follow either way

The full rules are in `CLAUDE.md`. Claude follows them automatically, but you're still responsible for the code you commit.

- **Never put secrets in code**: no API keys, passwords, or tokens, even temporarily. Use your team's secrets manager.
- **Check licenses** before adding a library. MIT, Apache 2.0, BSD, and ISC are approved. GPL, AGPL, LGPL, SSPL, and BSL need Legal approval.
- **Treat repository config as code that can run.** Be careful opening an unfamiliar repository that has a `.claude/` folder, `.mcp.json`, or hooks. Those files can change what Claude does.
- **Read before you approve.** When Claude asks to run a command, check what it does.
- **Fix the vulnerabilities in what you build.** Dependabot findings on your application are yours to fix.

## When something is blocked

1. Run `claude doctor` and `/status` to see which settings apply.
2. If an approved tool, host, or server is blocked, ask your administrator (managed) or update `.claude/settings.json` (self-managed).
3. Don't get around a block by running the command outside Claude or turning off the sandbox. Raise it instead.
