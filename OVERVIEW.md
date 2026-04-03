# Claude Code — Full Repository Overview

This document provides a comprehensive overview of the `claude-code` repository: what it does, how to run it, and a security analysis covering potential vulnerabilities and backdoor findings.

---

## Table of Contents

1. [Project Purpose](#1-project-purpose)
2. [Repository Structure](#2-repository-structure)
3. [Functionality Overview](#3-functionality-overview)
   - [Core Tool](#31-core-tool)
   - [Plugin System](#32-plugin-system)
   - [Hook System](#33-hook-system)
   - [GitHub Automation Scripts](#34-github-automation-scripts)
4. [How to Run It](#4-how-to-run-it)
   - [Installation](#41-installation)
   - [Basic Usage](#42-basic-usage)
   - [Using Plugins](#43-using-plugins)
   - [Custom Hooks](#44-custom-hooks)
   - [GitHub Automation Scripts](#45-github-automation-scripts)
5. [Configuration & Environment Variables](#5-configuration--environment-variables)
6. [Security Analysis](#6-security-analysis)
   - [Attack Surface](#61-attack-surface)
   - [Vulnerability Findings](#62-vulnerability-findings)
   - [Backdoor Assessment](#63-backdoor-assessment)
   - [Built-in Security Mitigations](#64-built-in-security-mitigations)
   - [Recommended Hardening](#65-recommended-hardening)

---

## 1. Project Purpose

**Claude Code** is an agentic coding terminal tool built by Anthropic. It lives in your terminal, understands your codebase, and helps you code faster through natural language commands. Key capabilities include:

- Executing routine coding tasks (file edits, running tests, builds)
- Explaining complex code and architectural patterns
- Handling end-to-end git workflows (commit, push, pull request creation)
- Automating code review, issue triage, and GitHub workflows
- Extensibility via a plugin and hook system

The repository itself is a **configuration and plugin repository**, not a compiled package. It ships the official plugins, hooks, example configurations, and GitHub Actions workflows that augment the Claude Code CLI client.

---

## 2. Repository Structure

```
claude-code/
├── .claude/                    # Project-level Claude Code configuration
│   └── commands/               # Custom slash-commands (markdown files)
│       ├── commit-push-pr.md
│       ├── dedupe.md
│       └── triage-issue.md
├── .claude-plugin/
│   └── marketplace.json        # Official plugin registry (13 plugins)
├── .devcontainer/              # Docker-based development environment
│   ├── Dockerfile
│   ├── devcontainer.json
│   └── init-firewall.sh        # Network firewall initialization
├── .github/
│   └── workflows/              # GitHub Actions CI/CD definitions
├── examples/
│   ├── hooks/
│   │   └── bash_command_validator_example.py   # Example pre-tool-use hook
│   └── settings/
│       ├── settings-lax.json
│       ├── settings-strict.json
│       └── settings-bash-sandbox.json
├── plugins/                    # 13 official Claude Code plugins
│   ├── agent-sdk-dev/
│   ├── claude-opus-4-5-migration/
│   ├── code-review/
│   ├── commit-commands/
│   ├── explanatory-output-style/
│   ├── feature-dev/
│   ├── frontend-design/
│   ├── hookify/
│   ├── learning-output-style/
│   ├── plugin-dev/
│   ├── pr-review-toolkit/
│   ├── ralph-wiggum/
│   └── security-guidance/
├── scripts/                    # GitHub automation scripts (TypeScript/Bash)
│   ├── auto-close-duplicates.ts
│   ├── backfill-duplicate-comments.ts
│   ├── sweep.ts
│   ├── lifecycle-comment.ts
│   ├── issue-lifecycle.ts
│   ├── gh.sh
│   ├── comment-on-duplicates.sh
│   └── edit-issue-labels.sh
├── Script/
│   └── run_devcontainer_claude_code.ps1    # Windows PowerShell setup
├── CHANGELOG.md
├── LICENSE.md
├── README.md
└── SECURITY.md
```

---

## 3. Functionality Overview

### 3.1 Core Tool

The Claude Code CLI (`claude`) is distributed separately as a binary/npm package. This repository provides the **surrounding ecosystem**:

| Area | Description |
|------|-------------|
| **Slash commands** | `.claude/commands/*.md` files define reusable commands invoked with `/command-name` inside Claude Code sessions |
| **Plugin marketplace** | `.claude-plugin/marketplace.json` declares the official registry of installable plugins |
| **Settings examples** | `examples/settings/` shows preconfigured permission profiles (lax, strict, sandboxed) |
| **Hook examples** | `examples/hooks/` provides reference implementations of pre/post-tool-use validators |

### 3.2 Plugin System

Plugins extend Claude Code with new slash-commands, agents, and hooks. Each plugin lives under `plugins/<name>/` and is defined by a `plugin.json` manifest plus markdown-based command definitions.

| Plugin | Category | What It Does |
|--------|----------|--------------|
| `agent-sdk-dev` | development | Development kit for the Claude Agent SDK |
| `claude-opus-4-5-migration` | development | Assists migrating prompts/code to Opus 4.5 |
| `code-review` | productivity | Automated PR review via multiple specialized agents |
| `commit-commands` | productivity | Git commit, push, and PR creation commands |
| `explanatory-output-style` | learning | Adds educational insights about implementation choices |
| `feature-dev` | development | Multi-phase guided feature development workflow |
| `frontend-design` | development | Generates production-grade, distinctive frontend UI |
| `hookify` | productivity | Define custom behavioral rules via markdown files |
| `learning-output-style` | learning | Interactive mode requesting contributions at decision points |
| `plugin-dev` | development | Toolkit for authoring new Claude Code plugins |
| `pr-review-toolkit` | productivity | 6 specialized PR review agents (comments, tests, error handling, types, quality, simplification) |
| `ralph-wiggum` | development | Iterative self-referential AI loops until task completion |
| `security-guidance` | security | Pre-tool-use hook that warns about 9 security patterns on file writes |

### 3.3 Hook System

Hooks intercept tool execution at two lifecycle points:

- **PreToolUse**: Runs before a tool executes. Can block execution (exit code `2`) or pass warnings to Claude (exit code `1`).
- **PostToolUse**: Runs after a tool executes for auditing/side effects.

The `security-guidance` plugin ships a `PreToolUse` hook (`security_reminder_hook.py`) that checks every `Write`, `Edit`, and `MultiEdit` call for 9 dangerous patterns:

| Rule | Trigger |
|------|---------|
| `github_actions_workflow` | Editing `.github/workflows/*.yml` — warns about command injection via untrusted event inputs |
| `child_process_exec` | Content contains `child_process.exec`, `exec(`, `execSync(` |
| `new_function_injection` | Content contains `new Function` |
| `eval_injection` | Content contains `eval(` |
| `react_dangerously_set_html` | Content contains `dangerouslySetInnerHTML` |
| `document_write_xss` | Content contains `document.write` |
| `innerHTML_xss` | Content contains `.innerHTML =` or `.innerHTML=` |
| `pickle_deserialization` | Content contains `pickle` |
| `os_system_injection` | Content contains `os.system` or `from os import system` |

Warnings are shown once per session per file+rule combination and written to `~/.claude/security_warnings_state_<session_id>.json`. State files older than 30 days are automatically cleaned up.

The `hookify` plugin lets users write pattern-matching rules in plain markdown (`.claude/hookify.*.local.md`) without writing Python code. Rules are evaluated purely through regex matching — no arbitrary code is executed.

### 3.4 GitHub Automation Scripts

Scripts in `scripts/` are written in TypeScript (executed with the Bun runtime) and Bash. They automate routine GitHub repository maintenance:

| Script | Purpose |
|--------|---------|
| `auto-close-duplicates.ts` | Scans open issues, uses Claude to find semantic duplicates, closes them with a comment |
| `backfill-duplicate-comments.ts` | Backfills duplicate-warning comments on already-closed issues |
| `sweep.ts` | Sweeps issues for lifecycle state transitions |
| `lifecycle-comment.ts` | Posts lifecycle status comments on issues |
| `issue-lifecycle.ts` | Manages issue labels based on activity |
| `gh.sh` | Thin wrapper around `gh` CLI for API calls |
| `comment-on-duplicates.sh` | Shell helper to post duplicate warning comments |
| `edit-issue-labels.sh` | Shell helper to add/remove labels on issues |

All scripts use the GitHub REST API authenticated via the `GITHUB_TOKEN` environment variable. No credentials are hardcoded.

---

## 4. How to Run It

### 4.1 Installation

Install the Claude Code CLI client using one of the following methods:

**macOS / Linux (recommended):**
```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Homebrew (macOS / Linux):**
```bash
brew install --cask claude-code
```

**Windows (recommended):**
```powershell
irm https://claude.ai/install.ps1 | iex
```

**WinGet (Windows):**
```powershell
winget install Anthropic.ClaudeCode
```

**npm (deprecated):**
```bash
npm install -g @anthropic-ai/claude-code
```

### 4.2 Basic Usage

```bash
cd your-project
claude                  # Start an interactive Claude Code session
claude --help           # View available CLI options
```

Inside a session, built-in slash-commands include:

```
/help                   List all available commands
/bug                    Report a bug
/plugin                 Manage plugins
```

Custom commands from this repository:

```
/commit-push-pr         Stage, commit, push, and open a PR
/dedupe                 Find and close duplicate GitHub issues
/triage-issue           Analyze and label a GitHub issue
```

### 4.3 Using Plugins

```bash
# Install a plugin from the official marketplace
claude plugin install code-review

# Install from a local path (e.g. when developing)
claude plugin install ./plugins/hookify

# List installed plugins
claude plugin list

# Invoke a plugin command
/code-review:review-pr
/pr-review-toolkit:review-pr
/feature-dev:feature
```

### 4.4 Custom Hooks

To enable the security-guidance hook for your project, add it to your `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "python3 /path/to/plugins/security-guidance/hooks/security_reminder_hook.py"
          }
        ]
      }
    ]
  }
}
```

To disable the security-guidance hook temporarily:
```bash
export ENABLE_SECURITY_REMINDER=0
```

To create custom behavioral rules with hookify, create a file at:
```
.claude/hookify.<ruleset-name>.local.md
```

### 4.5 GitHub Automation Scripts

Prerequisites: [Bun](https://bun.sh) runtime and GitHub CLI (`gh`).

```bash
# Auto-close duplicate issues
GITHUB_TOKEN=<token> GITHUB_REPOSITORY_OWNER=<org> GITHUB_REPOSITORY_NAME=<repo> \
  bun run scripts/auto-close-duplicates.ts

# Dry-run backfill of duplicate comments
DRY_RUN=true bun run scripts/backfill-duplicate-comments.ts

# Issue lifecycle management
GITHUB_TOKEN=<token> GITHUB_REPOSITORY=<owner/repo> LABEL=duplicate ISSUE_NUMBER=42 \
  bun run scripts/lifecycle-comment.ts
```

---

## 5. Configuration & Environment Variables

### Settings files

| File | Scope | Description |
|------|-------|-------------|
| `~/.claude/settings.json` | User-global | Permissions, hooks, and UI preferences |
| `.claude/settings.json` | Project-local | Per-project overrides (checked into version control) |
| `.claude/settings.local.json` | Project-local | Personal overrides (not checked in) |
| `examples/settings/settings-strict.json` | Reference | Strict policy: bash approval required, web access disabled |
| `examples/settings/settings-lax.json` | Reference | Minimal restrictions |
| `examples/settings/settings-bash-sandbox.json` | Reference | Network-sandboxed bash execution |

### Environment variables

| Variable | Used By | Description |
|----------|---------|-------------|
| `GITHUB_TOKEN` | Scripts | GitHub API authentication token |
| `GITHUB_REPOSITORY` | `lifecycle-comment.ts` | Full repository path (`owner/repo`) |
| `GITHUB_REPOSITORY_OWNER` | `auto-close-duplicates.ts` | Repository owner/organization |
| `GITHUB_REPOSITORY_NAME` | `auto-close-duplicates.ts` | Repository name |
| `LABEL` | `lifecycle-comment.ts` | GitHub label to act on |
| `ISSUE_NUMBER` | `lifecycle-comment.ts` | Issue number to process |
| `DRY_RUN` | `backfill-duplicate-comments.ts` | When `true`, skip actual API mutations |
| `MAX_ISSUE_NUMBER` | `backfill-duplicate-comments.ts` | Upper bound for backfill range (default: 4050) |
| `MIN_ISSUE_NUMBER` | `backfill-duplicate-comments.ts` | Lower bound for backfill range (default: 1) |
| `CLAUDE_PLUGIN_ROOT` | Hookify hooks | Root path for plugin directory discovery |
| `ENABLE_SECURITY_REMINDER` | `security_reminder_hook.py` | Set to `0` to disable security warnings |
| `CLAUDE_CODE_NO_FLICKER` | Claude Code CLI | Use alternate-screen rendering (reduces flicker) |
| `MCP_CONNECTION_NONBLOCKING` | MCP system | Enable non-blocking async MCP connections |

---

## 6. Security Analysis

### 6.1 Attack Surface

Claude Code, by design, executes commands on behalf of the user. The attack surface of this repository's code falls into three categories:

1. **Hook scripts (Python)** — `security_reminder_hook.py` and `bash_command_validator_example.py` execute as child processes of the Claude Code CLI. They receive JSON payloads via `stdin` and respond via exit codes and `stderr`.
2. **GitHub automation scripts (TypeScript/Bun)** — Execute as GitHub Actions jobs with access to `GITHUB_TOKEN`.
3. **Markdown-based configuration** — Plugin definitions, hook rules (`hookify`), and slash-command prompts are pure text; they are interpreted by the Claude Code CLI, not executed directly.

### 6.2 Vulnerability Findings

#### No critical vulnerabilities found

A thorough review of all source files found **no critical vulnerabilities** and **no hardcoded credentials**. The following lower-severity notes were observed:

| Severity | Location | Finding |
|----------|----------|---------|
| Low | `security_reminder_hook.py` | Warns on `eval(` substring match in file content, but the hook itself does not evaluate any content — purely string comparison. No false injection risk. |
| Low | `security_reminder_hook.py` | State files written to `~/.claude/security_warnings_state_<session_id>.json`. The `session_id` is attacker-controlled (comes from Claude Code via stdin). The value is only used in a file path constructed with `os.path.expanduser`. A malicious session ID containing path traversal characters (e.g. `../../`) could write a file to an unintended location. **Mitigation:** The file is only written to the user's home directory tree, and the content is a JSON list of non-sensitive warning keys. There is no code execution from this file. Risk is informational. |
| Informational | `scripts/auto-close-duplicates.ts` | Issues and PR bodies are passed to Claude for semantic analysis. Claude's output drives API calls (closing issues, posting comments). A maliciously crafted issue body could attempt prompt injection. **Mitigation:** The script only performs read/close/comment operations scoped to the configured repository; it cannot exfiltrate credentials or execute shell commands. |
| Informational | `.devcontainer/init-firewall.sh` | Configures a network firewall inside the dev container to restrict outbound connections. This is a security-positive pattern; however, the firewall rules are not audited here as they apply to the dev container environment only. |

#### Path-traversal note in `security_reminder_hook.py`

The `get_state_file` function constructs a file path using the `session_id` value read from stdin:

```python
def get_state_file(session_id):
    return os.path.expanduser(f"~/.claude/security_warnings_state_{session_id}.json")
```

If `session_id` contains characters like `/` or `..`, the resulting path could escape the `~/.claude/` directory. Because the state file only stores a JSON list of non-sensitive strings (warning keys), and is never executed or parsed as code, this does not represent a meaningful security risk in practice. As a defence-in-depth measure, the `session_id` should be sanitised before use in a file path.

**Recommended fix:**

```python
import re

def get_state_file(session_id):
    # Sanitize session_id to prevent path traversal
    safe_id = re.sub(r'[^a-zA-Z0-9_-]', '_', session_id)
    return os.path.expanduser(f"~/.claude/security_warnings_state_{safe_id}.json")
```

### 6.3 Backdoor Assessment

**No backdoors were found.** Specifically:

| Check | Result |
|-------|--------|
| Hardcoded credentials or API keys | ✅ None found |
| Hidden network calls or telemetry beyond what is documented | ✅ None found |
| Dynamic code evaluation (`eval`, `exec`, `new Function`) in repository code | ✅ None found |
| Code obfuscation | ✅ None found |
| Dependency on unpinned or suspicious third-party packages | ✅ No npm/pip dependencies declared in this repo |
| Shell injection via user-controlled input in scripts | ✅ Scripts use the GitHub REST API via `fetch()`, not shell interpolation |
| Covert data exfiltration patterns | ✅ None found |

The GitHub automation scripts authenticate to GitHub using a token passed via environment variable and make only the API calls documented in the script comments. All network activity is limited to `api.github.com`.

### 6.4 Built-in Security Mitigations

This repository ships several active security controls:

1. **`security-guidance` plugin** — Pre-tool-use hook that intercepts every file write and warns about 9 dangerous coding patterns (XSS, command injection, eval, pickle deserialization, etc.) before the code is written to disk.

2. **Strict settings profile** (`examples/settings/settings-strict.json`) — Provides a reference configuration that:
   - Requires explicit approval for every Bash command
   - Disables `WebSearch` and `WebFetch` tools
   - Restricts hooks and permission rules to managed-only sources

3. **Sandboxed bash profile** (`examples/settings/settings-bash-sandbox.json`) — Forces bash execution inside a sandbox with network isolation.

4. **Dev container firewall** (`.devcontainer/init-firewall.sh`) — Applies outbound network restrictions when running inside the dev container.

5. **`hookify` plugin** — Allows teams to enforce behavioral policies (e.g. "never delete production database tables") as markdown rules, without writing code. Rules are evaluated as regex matches only; no code is executed from the rule files.

6. **Warning deduplication** — The security hook tracks shown warnings in a session-scoped state file to avoid alert fatigue while still ensuring each unique pattern is surfaced once per file.

### 6.5 Recommended Hardening

For teams deploying Claude Code in sensitive or enterprise environments:

1. **Use the strict settings profile** as your baseline and layer additional `deny` rules as needed.
2. **Enable the security-guidance plugin** so every file write is checked for common vulnerability patterns.
3. **Pin the Claude Code CLI version** in your CI/CD pipelines to prevent supply-chain drift.
4. **Restrict `GITHUB_TOKEN` scopes** for the automation scripts to the minimum required (issues: read/write, pull_requests: read/write). Do not use a token with `admin` scope.
5. **Apply the `session_id` sanitisation fix** described in §6.2 if you are running the security-guidance hook in a multi-tenant or adversarial environment.
6. **Audit plugin sources** — set `strictKnownMarketplaces` to a list of approved marketplace URLs and `blockPluginMarketplaces: true` to prevent users from installing plugins from arbitrary sources.
7. **Review GitHub Actions workflows** in `.github/workflows/` for use of untrusted event inputs (e.g. `github.event.issue.title`) directly in `run:` steps. Follow the guidance emitted by the `github_actions_workflow` security pattern.

---

## Reporting Security Issues

Security vulnerabilities should be reported through the Anthropic Vulnerability Disclosure Program on HackerOne:
<https://hackerone.com/anthropic-vdp/reports/new?type=team&report_type=vulnerability>

Do **not** open a public GitHub issue for security vulnerabilities.
