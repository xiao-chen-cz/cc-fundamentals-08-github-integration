# Claude Code Extension System: Spec Sheet

> A reference guide explaining the difference between Hooks, Skills (/commands), MCPs, Plugins, and built-in /commands.

---

## Overview

Claude Code can be extended and customized through five distinct mechanisms. Each serves a different purpose and operates at a different layer of the system.

| Mechanism | Purpose | Configured In | Who Triggers It |
|-----------|---------|---------------|-----------------|
| **Hooks** | Run shell commands on events | `settings.json` | System (automatic) |
| **Skills** | Custom slash command prompts | `.claude/commands/` | User (manual) |
| **MCPs** | Add tools via external servers | `settings.json` | Claude (automatic) |
| **Plugins** | IDE/editor integrations | IDE marketplace | User (install once) |
| **/commands** | Built-in CLI control commands | Built into Claude Code | User (manual) |

---

## Hooks

**What they are**: Shell commands that execute automatically in response to specific lifecycle events.

**How they work**: Claude Code fires events at key moments (before a tool runs, after Claude responds, when a session starts, etc.). You can attach shell commands to these events in your settings.

**Configured in**: `~/.claude/settings.json` or `.claude/settings.json`

**Example events**:
- `PreToolUse` — runs before Claude uses a tool (e.g., Bash, Edit, Write)
- `PostToolUse` — runs after a tool completes
- `Notification` — fires when Claude wants to alert you
- `Stop` — fires when Claude finishes a task

**Example use cases**:
- Auto-format code after every file edit
- Log all tool usage to a file
- Play a sound when Claude finishes a long task
- Block dangerous commands (e.g., deny `rm -rf` in Bash)
- Send a desktop notification when work is done

**Example config**:
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [{ "type": "command", "command": "prettier --write $CLAUDE_TOOL_INPUT_FILE_PATH" }]
      }
    ]
  }
}
```

**Key distinction**: Hooks are **reactive** — they run automatically without user input. You define them once and they fire silently in the background.

---

## Skills (Custom Slash Commands)

**What they are**: User-defined prompt templates that can be invoked with `/skill-name`.

**How they work**: You create a Markdown file in `.claude/commands/`. The filename becomes the command name. When you type `/filename`, Claude Code reads that Markdown file and uses it as a structured prompt.

**Configured in**: `.claude/commands/*.md` (project-level) or `~/.claude/commands/*.md` (global)

**Example skills in this repo**:
- `/EA-prime` — prompts Claude to understand the codebase
- `/EA-handoff` — saves session state for later
- `/EA-pickup` — resumes from a previous handoff
- `/EA-install` — installs commands globally

**Example use cases**:
- `/commit` — run a standardized commit workflow
- `/review` — trigger a code review checklist
- `/deploy` — walk through a deployment checklist
- `/standup` — summarize recent git changes as a standup update

**Example file** (`.claude/commands/commit.md`):
```markdown
Review all staged changes and write a conventional commit message.
Check for: breaking changes, new features, bug fixes.
Format: type(scope): description
```

**Key distinction**: Skills are **on-demand** prompt shortcuts. They encode repeatable workflows as reusable commands that you invoke manually. They don't add new tools — they guide Claude's behavior with pre-written instructions.

---

## MCPs (Model Context Protocol Servers)

**What they are**: External processes that provide additional tools to Claude via a standardized protocol.

**How they work**: An MCP server is a separate program (local or remote) that exposes a set of tools. Claude Code connects to it at startup and these tools become available to Claude alongside built-in tools like Bash, Read, and Edit.

**Configured in**: `~/.claude/settings.json` or `.claude/settings.json`

**Example MCP servers**:
- `github` — tools for GitHub API operations (create PRs, fetch issues, post comments)
- `postgres` — tools for querying a database
- `filesystem` — expanded file system access
- `puppeteer` — browser automation tools
- `slack` — tools for sending Slack messages

**Example config**:
```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "your-token" }
    }
  }
}
```

**Key distinction**: MCPs **expand what Claude can do**. They add new capabilities (tools) that Claude can call. Without the MCP, those tools don't exist. Skills tell Claude *how* to use existing tools; MCPs give Claude *new* tools entirely.

---

## Plugins

**What they are**: Integrations that embed Claude Code inside an IDE or editor.

**How they work**: IDE plugins wrap the Claude Code CLI inside a native editor UI. They provide features like inline diffs, file tree access, terminal integration, and keybindings — all without leaving your editor.

**Installed via**: IDE marketplace (VS Code Extensions, JetBrains Marketplace, etc.)

**Available plugins**:
- **VS Code** — Claude Code extension in the VS Code Marketplace
- **JetBrains** — Plugin for IntelliJ, WebStorm, PyCharm, etc.
- **Vim/Neovim** — Community integrations

**What plugins provide**:
- Open Claude Code in a side panel inside your editor
- Click to accept/reject diffs visually
- Automatic file context from whatever you have open
- Keyboard shortcuts for common operations
- Better integration with editor terminals

**Key distinction**: Plugins are **editor integrations** — they change *where* you interact with Claude Code (inside your editor vs. a standalone terminal). They don't add new tools or behaviors; they provide a better UX for the same underlying Claude Code engine.

---

## Built-in /commands

**What they are**: Built-in commands that control Claude Code's behavior, session, and configuration.

**How they work**: Type `/command` in the Claude Code prompt. These are hardcoded into Claude Code itself (not files you create). They perform meta-operations on the session or tool.

**Available built-in commands** (selection):

| Command | What it does |
|---------|-------------|
| `/help` | Show help and available commands |
| `/clear` | Clear conversation history |
| `/compact` | Summarize conversation to save context |
| `/memory` | View and edit Claude's memory files |
| `/config` | View or change settings |
| `/cost` | Show token usage and cost for this session |
| `/doctor` | Check Claude Code installation health |
| `/review` | Review code changes |
| `/vim` | Toggle vim keybinding mode |
| `/fast` | Toggle fast (Sonnet) mode |
| `/logout` | Log out of your Anthropic account |
| `/quit` | Exit Claude Code |

**Key distinction**: Built-in `/commands` are **session and tool controls**. They manage Claude Code itself — history, context, settings, cost. You cannot create new built-in commands (that's what Skills are for).

---

## Side-by-Side Comparison

```
┌─────────────────┬──────────────┬────────────────┬──────────────┬─────────────────┐
│                 │   HOOKS      │   SKILLS       │   MCPs       │   PLUGINS       │
├─────────────────┼──────────────┼────────────────┼──────────────┼─────────────────┤
│ Triggered by    │ System event │ User types /   │ Claude calls │ User installs   │
│                 │ (automatic)  │ slash command  │ tool         │ in IDE          │
├─────────────────┼──────────────┼────────────────┼──────────────┼─────────────────┤
│ What it does    │ Runs shell   │ Injects a      │ Adds new     │ Wraps Claude    │
│                 │ command      │ prompt template│ tools        │ in editor UI    │
├─────────────────┼──────────────┼────────────────┼──────────────┼─────────────────┤
│ Configured in   │ settings.json│ .claude/       │ settings.json│ IDE marketplace │
│                 │              │ commands/*.md  │              │                 │
├─────────────────┼──────────────┼────────────────┼──────────────┼─────────────────┤
│ Runs as         │ Shell process│ Prompt context │ Separate     │ Editor extension│
│                 │              │ injection      │ server       │                 │
├─────────────────┼──────────────┼────────────────┼──────────────┼─────────────────┤
│ Extends         │ Lifecycle    │ Prompt         │ Tool         │ UI/UX           │
│                 │ behavior     │ behavior       │ capabilities │                 │
└─────────────────┴──────────────┴────────────────┴──────────────┴─────────────────┘
```

---

## Mental Model

Think of Claude Code as a car:

- **Hooks** = cruise control and sensors — automatic systems that fire without your input
- **Skills** = the radio presets — shortcuts you set up once and invoke with a button press
- **MCPs** = aftermarket accessories — new hardware capabilities bolted on (GPS, backup camera)
- **Plugins** = different car bodies — same engine, but you're driving a coupe vs. a truck cab
- **Built-in /commands** = dashboard controls — steering, windows, mirrors; the standard controls that ship with every car

---

## Quick Reference: When to Use What

| Goal | Use |
|------|-----|
| Auto-format files after editing | Hook (PostToolUse on Edit) |
| Standardize a commit message workflow | Skill (/commit) |
| Give Claude access to a database | MCP (database server) |
| Work in VS Code instead of terminal | Plugin (VS Code extension) |
| Clear conversation history | Built-in /command (/clear) |
| Block dangerous shell commands | Hook (PreToolUse on Bash) |
| Encode a code review checklist | Skill (/review) |
| Post GitHub comments via Claude | MCP (github MCP server) |
| See how many tokens you've used | Built-in /command (/cost) |
