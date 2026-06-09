# Claude Code Extension Mechanisms: Hooks, Slash Commands, Skills, MCPs, and Plugins

This spec explains the five primary extension and customization mechanisms available in Claude Code, how they differ, and when to use each one.

---

## Overview

| Mechanism | Who controls it | When it runs | Primary purpose |
|---|---|---|---|
| Hooks | Harness/system | Automatically on events | Side effects, enforcement, automation |
| Slash Commands | User (via typing) | On user invocation | User-triggered workflows |
| Skills | Claude (via Skill tool) | When matched by Claude or user | Reusable, composable capabilities |
| MCPs | External server | When Claude calls a tool | Extending Claude's toolset |
| Plugins | IDE/environment | On environment events | Editor and environment integration |

---

## Hooks

Hooks are **shell commands configured in `settings.json`** that execute automatically in response to specific lifecycle events. They run outside of Claude's control — the harness executes them, not Claude itself.

**When they run:** On events like `PreToolUse`, `PostToolUse`, `Stop`, `Notification`, etc.

**What they're for:**
- Enforcing policies (e.g., block certain tools automatically)
- Triggering side effects after Claude finishes (e.g., send a Slack notification)
- Automating recurring setup or teardown (e.g., run a linter after every file edit)

**Key characteristic:** Hooks are **unconditional and automatic**. Claude cannot skip them, and the user does not need to type anything to trigger them. They are the right tool for "always do X when Y happens."

**Example use case:** "After every file edit, run `prettier --write` on the changed file."

**Configuration:** Defined in `.claude/settings.json` or `~/.claude/settings.json` under the `hooks` key.

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

---

## Slash Commands

Slash commands are **user-typed directives** prefixed with `/` that trigger specific behaviors in the Claude Code interface. Some are built-in; others are custom and backed by a skill file.

**When they run:** When a user explicitly types `/<command>` in the chat input.

**What they're for:**
- Triggering a workflow the user wants to initiate (e.g., `/review`, `/help`)
- User-controlled, ad-hoc actions
- Providing a discoverable, consistent interface for common tasks

**Key characteristic:** Slash commands are **user-initiated**. Nothing happens automatically — the user must type the command. Built-in slash commands like `/help`, `/clear`, `/config`, and `/fast` are provided by Claude Code itself. Custom slash commands are defined as skill files and surfaced to users via the `/` prefix.

**Example use case:** A user types `/code-review` to trigger a code review of the current diff.

**Relationship to skills:** When a custom slash command is invoked, Claude Code resolves it to the corresponding skill file (e.g., `.claude/commands/code-review.md`) and executes it via the `Skill` tool.

---

## Skills

Skills are **named, reusable capability definitions** stored as Markdown files in `.claude/commands/`. They serve as both the backing implementation for custom slash commands and as composable building blocks that Claude can invoke programmatically.

**When they run:** Either when a user types the matching slash command, or when Claude itself calls the `Skill` tool (e.g., during a multi-step task when Claude determines a skill is relevant).

**What they're for:**
- Encapsulating domain-specific workflows in a reusable, shareable format
- Providing Claude with structured instructions for complex tasks
- Enabling multi-step automation that Claude orchestrates (e.g., `EA-handoff`, `deep-research`)

**Key characteristic:** Skills are **Claude-readable instruction files**. They are more than macros — they can contain rich prompting, context, and structured steps that guide Claude's behavior for a specific capability. Claude can invoke skills autonomously when it determines they match the task at hand.

**File location:** `.claude/commands/<skill-name>.md`

**Example use case:** A `security-review.md` skill that contains detailed instructions for performing a security audit, invoked via `/security-review` or by Claude when asked to review code for vulnerabilities.

**Difference from slash commands:** Slash commands are the *user interface* (the `/` syntax). Skills are the *implementation* (the `.md` file). Every custom slash command is backed by a skill, but skills can also be invoked by Claude without user-typed slash commands.

---

## MCPs (Model Context Protocol Servers)

MCPs are **external processes that expose custom tools to Claude** via a standardized protocol. They extend Claude's built-in toolset with capabilities that connect to external systems, APIs, databases, or services.

**When they run:** When Claude calls an MCP-provided tool during a task (e.g., `mcp__github__create_pr`, `mcp__github_comment__update_claude_comment`).

**What they're for:**
- Giving Claude access to external systems (GitHub, Slack, Linear, databases, etc.)
- Adding domain-specific tools that Claude can discover and use
- Enabling Claude to take actions beyond its built-in file/shell capabilities

**Key characteristic:** MCPs are **separate server processes** that communicate with Claude Code over a defined protocol (stdio or HTTP). Claude discovers their tools at startup and can call them just like built-in tools. The tool schemas are loaded on demand.

**Example use case:** A `github` MCP server that exposes tools like `create_pull_request`, `list_issues`, and `post_comment`, allowing Claude to interact directly with the GitHub API.

**Configuration:** Defined in `.claude/settings.json` under `mcpServers`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-server-github"]
    }
  }
}
```

**Difference from hooks:** Hooks run on lifecycle events and are configured statically. MCP tools are called by Claude during a task based on what it needs to accomplish.

---

## Plugins

Plugins are **environment-level integrations** that extend Claude Code within a specific host environment — primarily IDE extensions (VS Code, JetBrains) or shell integrations.

**When they run:** On IDE or environment events (file open, editor focus change, terminal events, etc.), or passively as an always-on integration layer.

**What they're for:**
- Bringing Claude Code capabilities into the editor UI (e.g., inline suggestions, diff views)
- Providing context from the IDE to Claude (e.g., current cursor position, selected code)
- Integrating Claude into the developer's existing environment without switching to a terminal

**Key characteristic:** Plugins are **environment-specific** and operate at the level of the host application (VS Code, JetBrains). They are not scripts or config entries — they are installable extensions with their own lifecycles managed by the IDE.

**Example use case:** The Claude Code VS Code extension that adds a Claude panel to the editor sidebar, allows inline code suggestions, and lets developers invoke Claude on selected code without leaving the IDE.

**Difference from MCPs:** MCPs extend *what tools Claude can call*. Plugins extend *where and how users can interact with Claude Code* (the surface area). An IDE plugin may internally use MCP to communicate with Claude, but its purpose is UX integration, not tool extension.

---

## Decision Guide: Which Mechanism to Use?

| I want to… | Use |
|---|---|
| Automatically run something every time Claude edits a file | **Hook** |
| Let users trigger a workflow by typing `/something` | **Slash Command + Skill** |
| Give Claude a reusable set of instructions for a complex task | **Skill** |
| Let Claude call a third-party API or external service | **MCP** |
| Bring Claude into my IDE with a native UI | **Plugin** |

---

## How They Compose

These mechanisms are designed to work together:

1. A user types `/deep-research` (slash command) → Claude invokes the `deep-research` skill → the skill instructs Claude to use `WebSearch` and `WebFetch` tools (built-in) and an MCP-provided tool to save results → a hook runs after Claude stops to notify the team in Slack.

2. An IDE plugin (VS Code extension) captures selected code → passes it to Claude Code → Claude uses a `code-review` skill → results are displayed inline in the editor.

This layered architecture allows each mechanism to stay focused on its responsibility while composing cleanly with the others.
