# Claude Code: Commands, Hooks, Sub-Agents, and Skills

A reference spec comparing the four extension mechanisms in Claude Code.

---

## Overview

| Feature | Defined By | Invoked By | Runs As |
|---------|-----------|-----------|---------|
| **Commands** | Markdown file in `.claude/commands/` | User types `/command-name` | Claude reads and executes the prompt |
| **Hooks** | Shell commands in `settings.json` | Lifecycle events (automatic) | Subprocess on the host machine |
| **Sub-agents** | Built-in agent types | Claude uses the `Task` tool | Separate Claude instance |
| **Skills** | Registered command files | User types `/skill-name` or Claude detects intent | Claude executes via the `Skill` tool |

---

## Commands

### What They Are

Custom slash commands defined as Markdown files. They extend Claude's behavior with reusable, project-specific or global workflows.

### Where Defined

| Scope | Path |
|-------|------|
| Project | `.claude/commands/<name>.md` |
| Global | `~/.claude/commands/<name>.md` |

### File Format

```markdown
---
description: Brief description shown in /help
allowed-tools: Bash, Read, Write, Edit, Glob
model: sonnet
argument-hint: [optional hint shown to user]
---

# Command Title

Instructions Claude follows when this command is invoked.
```

### Frontmatter Fields

| Field | Purpose |
|-------|---------|
| `description` | Shown in `/help` listing |
| `allowed-tools` | Restricts which tools Claude may use |
| `model` | Override the model for this command |
| `argument-hint` | Hint displayed after the command name |

### How Invoked

User types `/command-name` (optionally with arguments) in the Claude Code interface.

### Use Cases

- Standardized project workflows (e.g., `/EA-prime` to understand a codebase)
- Repeatable processes like handoffs and pickups
- Onboarding steps that must be run the same way every time

### Example

```
/EA-prime
```

Claude reads `.claude/commands/EA-prime.md` and executes the instructions inside it — running `git ls-files`, reading the README, and producing a structured project summary.

---

## Hooks

### What They Are

Shell commands that run automatically at specific points in Claude's lifecycle. They are defined by the user (not Claude), and their output is fed back to Claude or used for side effects.

### Where Defined

In `settings.json` at the project or user level:

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'Prompt received'"
          }
        ]
      }
    ]
  }
}
```

### Lifecycle Events

| Event | When It Fires |
|-------|--------------|
| `UserPromptSubmit` | When the user submits a message |
| `PreToolUse` | Before Claude calls a tool |
| `PostToolUse` | After a tool call completes |
| `Stop` | When Claude finishes its response |

### How Invoked

Automatically — the user never types anything. The event fires, the shell command runs, and (depending on exit code) Claude may read the output or be blocked.

### Hook Behavior

- Exit code `0`: Success; stdout is shown to Claude as context
- Non-zero exit code: Claude is notified the hook blocked the action
- Claude can adjust its behavior based on hook feedback

### Use Cases

- Run a linter before every tool use and surface warnings to Claude
- Log all prompts for auditing
- Validate that generated code compiles before Claude proceeds
- Send notifications when Claude finishes a task

### Example

```json
"PreToolUse": [{
  "matcher": "Bash",
  "hooks": [{
    "type": "command",
    "command": "shellcheck $CLAUDE_TOOL_INPUT_COMMAND 2>&1 || true"
  }]
}]
```

---

## Sub-Agents

### What They Are

Separate Claude instances spawned by Claude using the `Task` tool. Each sub-agent runs with a specific purpose, its own context window, and a restricted tool set.

### How Invoked

Claude (not the user) calls the `Task` tool when a task matches a sub-agent's capabilities:

```
Task(subagent_type="Explore", prompt="Find all API endpoints in the codebase")
```

### Built-in Sub-Agent Types

| Type | Primary Use | Tools Available |
|------|------------|----------------|
| `Bash` | Terminal operations, git, shell commands | Bash |
| `Explore` | Codebase exploration, file and keyword search | All except Task, Edit, Write |
| `Plan` | Designing implementation strategies | All except Task, Edit, Write |
| `general-purpose` | Research, multi-step investigation | All tools |
| `claude-code-guide` | Questions about Claude Code itself | Glob, Grep, Read, WebFetch, WebSearch |
| `statusline-setup` | Configure the Claude Code status line | Read, Edit |

### Key Properties

| Property | Detail |
|----------|--------|
| Context | Each sub-agent has its own isolated context window |
| Concurrency | Multiple sub-agents can run in parallel |
| Communication | Returns a single result message to the parent |
| Resumption | Sub-agents can be resumed by ID |
| Background | Can run in the background; parent checks output later |

### Use Cases

- Parallelizing independent research tasks to save time
- Offloading deep codebase searches without consuming the main context
- Running specialized agents (Plan agent to design, Explore agent to find files)
- Separating concerns: one agent researches, another implements

### Example

```
Parent Claude uses Task tool:
  subagent_type: "Explore"
  prompt: "Find all files that handle authentication"

→ Explore agent runs Glob and Grep searches
→ Returns a report to parent
→ Parent continues with the findings
```

---

## Skills

### What They Are

Skills are the user-facing invocation layer for commands. When a user types `/skill-name`, Claude uses the `Skill` tool to execute the matching registered command. From the user's perspective, skills and commands look identical — the distinction is in how Claude routes the invocation internally.

### How Invoked

1. User types `/skill-name` in the chat
2. Claude detects the slash command pattern
3. Claude calls the `Skill` tool with the skill name
4. The Skill tool loads and executes the associated command

```
/EA-handoff
```

### How They Differ from Commands

| Dimension | Command | Skill |
|-----------|---------|-------|
| Definition | Markdown file in `.claude/commands/` | Same underlying file |
| Invocation path | Direct slash command | Routed through the `Skill` tool |
| Claude's perspective | Reads the file and follows instructions | Uses the `Skill` tool to delegate execution |
| Listed in system context | No | Yes — available skills are listed in the system prompt |
| Intent detection | Manual (`/command`) | Claude may invoke proactively if user intent matches |

### Use Cases

- All command use cases apply
- Skills can be invoked by Claude automatically when it detects user intent, without the user typing the slash command explicitly
- Useful for creating structured, domain-specific workflows (e.g., keybinding setup, session handoffs)

---

## Side-by-Side Comparison

| Dimension | Commands | Hooks | Sub-Agents | Skills |
|-----------|---------|-------|-----------|--------|
| **Who invokes** | User (slash command) | Automatic (lifecycle event) | Claude (Task tool) | User or Claude (Skill tool) |
| **Who defines** | User/developer | User/developer | Anthropic (built-in types) | User/developer (same as commands) |
| **Runs where** | Claude's context | Host shell subprocess | Separate Claude instance | Claude's context |
| **Has own context** | No | No | Yes | No |
| **Can run in parallel** | No | No (sequential per event) | Yes | No |
| **Receives tool access** | Configured via frontmatter | N/A (shell only) | Per agent type | Configured via frontmatter |
| **Trigger type** | Explicit user action | Automatic event | Claude decision | Explicit user action or Claude intent |
| **Output destination** | Claude's response | Fed back to Claude as context | Returned to parent Claude | Claude's response |
| **Primary purpose** | Reusable workflows | Side effects and validation | Parallel/specialized tasks | Reusable workflows with intent detection |

---

## Decision Guide

**Use a Command when:**
- You have a repeatable workflow you want to run with a slash command
- You want project-specific or global shortcuts for Claude

**Use a Hook when:**
- You want something to happen automatically without typing anything
- You need to validate, log, or modify behavior at specific lifecycle points

**Use a Sub-Agent when:**
- A task is independent enough to run in parallel with other work
- You want to isolate a large search or analysis from the main context
- You need a specialized Claude instance (planner, explorer, etc.)

**Use a Skill when:**
- Same as commands, but you want Claude to potentially invoke it based on user intent
- You want the command registered in the system-level available-skills list

---

*Reference: Claude Code documentation and `.claude/commands/` examples in this repo.*
