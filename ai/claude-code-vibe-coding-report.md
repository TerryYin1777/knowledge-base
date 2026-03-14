# Claude Code & Vibe Coding: A Practical Guide for 2025

**Audience:** Python/data engineering expert, newer to app development  
**Purpose:** Ship real things faster with Claude Code, not just experiment  
**Date:** March 2026

---

## Table of Contents

1. [What Is Claude Code — Architecture & How It Works](#1-what-is-claude-code--architecture--how-it-works)
2. [Core Features & Usage](#2-core-features--usage)
3. [Extension Points: Skills, MCP, Plugins, Hooks & More](#3-extension-points-skills-mcp-plugins-hooks--more)
4. [Productivity Tips & Tricks](#4-productivity-tips--tricks)
5. [Spec-Driven Development](#5-spec-driven-development) — incl. [OpenSpec](#openspec)
6. [Vibe Coding — State of the Art in 2025](#6-vibe-coding--state-of-the-art-in-2025)
7. [Recommended Workflow](#7-recommended-workflow)

---

## 1. What Is Claude Code — Architecture & How It Works

### The Essential Architecture: Two Separate Systems

The most important thing to understand about Claude Code before you write a single line — or rather, *ask* it to write a single line — is that **Claude Code and Claude are two completely different pieces of software**.

Claude Code is a command-line tool that runs locally on your machine. It's a client application, like `git` or `kubectl`. Claude (the actual AI model) runs on Anthropic's servers in the cloud.

Think of Claude Code as a browser, and Claude as a very smart website. The browser runs on your machine and sends requests. The website processes them and sends responses. The browser doesn't think — it executes. All the intelligence lives elsewhere.

This matters enormously:
- Your files never leave your machine unless Claude explicitly requests them
- You don't need local GPU hardware
- When Anthropic improves the model, you benefit immediately without reinstalling
- The CLI itself is lightweight and fast — the heavy computation is remote

### The Agent Loop: How Claude Code Thinks and Acts

When you type something like:
```
> Find all places where we're not handling database connection errors
```

Here's the actual sequence:
1. Claude Code packages your request + a **menu of available tools** into an API call
2. Claude (the model) receives the request, reasons about it, decides what tools it needs
3. Claude responds with a structured **tool call request** (e.g., "run this bash command")
4. Claude Code executes the tool on your local machine
5. Claude Code sends the results back to Claude
6. Claude reasons again: done, or need more info?
7. Loop until Claude returns a plain text response

Claude cannot touch your filesystem or terminal directly. It can only *request* that Claude Code do things. Claude Code is the hands; Claude is the brain.

This agent loop is fundamentally different from autocomplete tools like GitHub Copilot. Copilot sees your cursor position and completes a line. Claude Code understands your entire codebase, runs commands, reads error messages, and iterates until the job is done.

### Tool Use: What Claude Code Can Do

Claude Code's built-in tools include:

| Tool | What it does |
|------|-------------|
| `Read` | Read file contents |
| `Write` | Create or overwrite files |
| `Edit` | Make surgical edits to specific sections |
| `Bash` | Run shell commands, scripts, tests |
| `Glob` | List files matching patterns |
| `Grep` | Search file contents |
| `WebFetch` | Fetch content from URLs |
| `Task` | Spawn a subagent for a parallel task |
| `TodoRead/Write` | Manage task checklists |

You can extend these with MCP servers (covered in section 3).

### Context Window Management — Large Codebases

Context is the critical constraint. Claude has a large but finite context window (200K tokens for Claude 3.5/4 Sonnet, 1M+ for some models). Your entire codebase won't fit. Claude Code manages this by:

1. **Not loading everything upfront.** It reads files on-demand as needed by the task.
2. **CLAUDE.md as a map.** The CLAUDE.md file tells Claude what exists and where, so it can request the right files.
3. **`/compact`** — compresses conversation history when context gets large, preserving summaries of what happened.
4. **`/clear`** — wipes the session entirely, starting fresh.

For a large codebase (say, a FastAPI backend + React frontend + Airflow DAGs), you don't dump everything in at once. Instead, structure your CLAUDE.md so Claude knows the layout, and be specific in your requests: "Look at `src/pipeline/ingestion.py` and the error at line 47."

The key insight from experienced users: **a focused session with precise file references beats a "please read everything" approach every time.**

### The Role of CLAUDE.md / AGENTS.md as Persistent Memory

Claude Code wakes up fresh every session. CLAUDE.md is how you give it permanent memory.

**Where CLAUDE.md files live:**
- `~/.claude/CLAUDE.md` — global instructions (coding style, preferences)
- `<project_root>/CLAUDE.md` — project-level context
- `<subdirectory>/CLAUDE.md` — subdirectory-specific context (e.g., `/frontend/CLAUDE.md`)

All of these get loaded automatically when you work in that directory. Claude reads them as text and includes them in the context. The AI interprets the instructions — not the CLI tool.

**AGENTS.md** is an open standard supported by tools like Cursor, Windsurf, and others. Claude Code uses CLAUDE.md specifically. If you want cross-tool compatibility, use AGENTS.md; for pure Claude Code work, CLAUDE.md is the way.

One important nuance: you can write conditional instructions in CLAUDE.md that Claude actually understands. Example:

```markdown
# Security Guidelines
When implementing any authentication endpoint, read SECURITY.md first.
For non-auth changes, skip SECURITY.md unless explicitly asked.
```

Claude understands "authentication endpoint" semantically — it's not keyword matching, it's comprehension. This makes CLAUDE.md dramatically more powerful than simple config files.

### How It Differs from ChatGPT/Copilot

| | ChatGPT | GitHub Copilot | Claude Code |
|---|---|---|---|
| Interface | Web/API chat | IDE inline completion | CLI agent |
| Reads your codebase | Manual paste | Current file | Yes, on demand |
| Runs commands | No | No | Yes |
| Writes files | No | Suggestions only | Yes |
| Iterates autonomously | No | No | Yes |
| Memory across sessions | Threads only | No | Via CLAUDE.md |

ChatGPT is a conversation partner. Copilot is autocomplete on steroids. Claude Code is an autonomous agent that can explore your project, run your tests, fix the failures, and commit the result. The mental model shift is significant.

### Multi-Agent: Orchestrator + Subagent Model

Claude Code supports a multi-agent model where one Claude instance (orchestrator) spawns multiple parallel subagents using the `Task` tool. This is now a core feature, not a hack.

Real example from a developer migrating a storage layer:
1. Orchestrator receives the task: "Migrate from SQLite to IndexedDB"
2. Spawns 5 parallel research agents, each investigating a different aspect of the target library
3. Research agents report back their findings
4. Orchestrator synthesizes a spec
5. Spawns implementation agents per task, each working in isolation

Each subagent has a clean context window — no pollution from other tasks. The orchestrator manages coordination and commits. This pattern enables work that would overflow a single context window to be broken into parallelizable pieces.

The task system persists tasks to disk in `.claude/tasks/{session-id}/` as JSON files, so work survives session restarts. This is a major quality-of-life improvement over earlier approaches.

---

## 2. Core Features & Usage

### CLI Interface: Key Commands and Modes

Install Claude Code:
```bash
npm install -g @anthropic-ai/claude-code
```

Basic usage — just launch it in your project directory:
```bash
cd ~/myproject
claude
```

Key flags:
```bash
# Start a fresh session
claude

# Resume the previous session (critical for long work)
claude --resume

# Run a single prompt non-interactively (headless)
claude -p "Run the test suite and fix any failures"

# Specify a model
claude --model claude-opus-4-6

# Print without streaming (for scripting)
claude --no-stream -p "List all API endpoints in this codebase"
```

For most day-to-day work, you'll just `cd` into your project and type `claude`. The session-resumption (`--resume`) is underrated — use it constantly to maintain context across breaks.

### Slash Commands — Built-in and Most Useful

Type `/` in Claude Code to see all commands. The most important ones:

| Command | What it does |
|---------|-------------|
| `/help` | Shows all available commands |
| `/init` | Creates a CLAUDE.md for the current project |
| `/clear` | Wipes the session (fresh start) |
| `/compact` | Summarizes conversation history to free context |
| `/memory` | Opens CLAUDE.md for editing |
| `/context` | Shows current context window usage |
| `/agents` | Manages subagents |
| `/plan` | Enters planning mode (or press Shift+Tab twice) |
| `/review` | Triggers a code review of recent changes |
| `/pr` | Creates a pull request |
| `/diff` | Shows a diff of current changes |

**`/compact` vs `/clear`:** Use `/compact` when you want to continue the current task but are running low on context — it summarizes what happened so far and keeps going. Use `/clear` when the current approach has gone wrong and you want a completely fresh attempt. Starting over is often faster than trying to fix a derailed conversation.

### Extended Thinking Mode

One of the most powerful (and underused) features. You can invoke different levels of reasoning depth:

```
> think: what's the best way to structure the authentication for this API?

> think hard: analyze all the edge cases in this rate limiting implementation

> ultrathink: design a migration strategy for moving from this monolith to microservices
```

These phrases are mapped directly to increasing computational "thinking budget." `ultrathink` burns more tokens but gives you deeper, more careful analysis. Use it for architecture decisions, complex debugging, security reviews — problems where you want Claude to genuinely reason through the solution rather than pattern-match to something common.

The hierarchy: `think` < `think hard` < `think harder` < `ultrathink`

Don't throw `ultrathink` at everything — it costs more and slows responses. Save it for decisions that actually matter.

### Plan Mode

Press `Shift+Tab` twice to enter Plan Mode. In this mode, Claude analyzes your codebase and creates an implementation strategy — but **cannot change any files**. It's read-only architect mode.

Why this is critical: without planning, Claude jumps straight to implementation and often makes locally reasonable choices that are globally incoherent. Plan Mode forces the "think before you code" discipline.

Use Plan Mode:
- **Always** at the start of any non-trivial feature
- Before any refactoring that touches multiple files
- When debugging an issue you don't fully understand
- When making architecture decisions

The workflow:
1. Enter Plan Mode (Shift+Tab×2)
2. Describe what you want to achieve
3. Let Claude read the codebase and propose a plan
4. Review and critique the plan — ask questions, push back on bad ideas
5. Exit Plan Mode, let it execute

### Reading/Writing Files, Running Bash

Claude Code can directly read and write files, which means you can have conversations like:

```
> Read the requirements.txt and generate a pyproject.toml equivalent

> Run the test suite and show me only the failures

> Look at src/api/auth.py and identify any security issues

> Refactor the database queries in db/queries.py to use SQLAlchemy's ORM pattern
```

For bash commands, Claude Code will ask permission before running anything destructive. You can configure trust levels in your CLAUDE.md or set an `--allow-tools` flag for automated pipelines.

### Git Integration

Claude Code has strong git awareness built in:

```
> Create a branch called feature/user-auth and implement JWT authentication
> Commit all current changes with a meaningful commit message
> Show me what changed since the last commit
> Create a PR for this branch against main
```

It reads git history, understands diffs, can cherry-pick changes, and creates conventional commit messages. The PR workflow is genuinely good — it can draft PR descriptions summarizing what changed and why.

**Best practice:** Always work on feature branches. Never let Claude work on `main` directly. If it goes wrong (and sometimes it will), you want a branch you can delete without consequence.

### Prompt Engineering for Agentic Coding

This is not chatting with a smart person — it's directing an agent. The prompting style differs:

**Be specific about scope:**
```
❌ "Fix the authentication"
✅ "In src/auth/handlers.py, the login endpoint isn't validating JWT expiry. Add expiry validation before the user lookup."
```

**Reference files explicitly:**
```
✅ "Look at @src/pipeline/transform.py and @tests/test_transform.py"
```

**State acceptance criteria:**
```
✅ "The function should handle null inputs by returning an empty dict, not raising. Add tests for this case."
```

**Use questions to refine before coding:**
```
✅ "Before implementing the user auth system, ask me questions about requirements."
```

This last pattern is powerful. Prompting Claude to ask you questions before it starts catches assumptions you didn't know you were making.

---

## 3. Extension Points: Skills, MCP, Plugins, Hooks & More {#3-extension-points-skills-mcp-plugins-hooks--more}

Claude Code has **six distinct ways to extend its behavior**. Most developers conflate them or don't know all six exist. Each answers a different question:

| Question | Mechanism |
|----------|-----------|
| What should Claude *know how to do*? | **Skills / Slash Commands** (same thing) |
| What external tools can Claude *access*? | **MCP** |
| Who does the work? | **Sub-agents** |
| When should things happen *automatically*? | **Hooks** |
| How do you *package and share* all of the above? | **Plugins** |

Understanding which one to reach for — and why — is one of the highest-leverage things you can learn.

---

### Skills = Slash Commands (They're the Same Thing)

> **Terminology alert:** "Skills" and "custom slash commands" are the same thing in Claude Code. Both refer to markdown files in `.claude/commands/` that you invoke with `/name`. Official docs now prefer "skills" but you'll see both terms used interchangeably everywhere. Don't let this confuse you.

> **⚠️ OpenClaw Skills ≠ Claude Code Skills** — OpenClaw has its own skill system (SKILL.md in a different structure, used by OpenClaw agents). These are completely separate concepts that share a name. This section is about Claude Code skills only.

**Skills teach Claude how to perform tasks.** A skill is a markdown file (optionally with YAML frontmatter) that lives in `.claude/commands/`. Claude invokes it with `/skill-name`, or can auto-trigger it when the description matches the task.

**Basic skill structure:**

```
.claude/commands/
├── code-review.md      → /code-review
├── pr.md               → /pr
└── deploy/
    └── staging.md      → /deploy:staging
```

```markdown
---
description: Security-focused code review following OWASP guidelines
allowed-tools: Read, Grep, Glob
---

When reviewing code:
1. Check for injection vulnerabilities (SQL, XSS, command injection)
2. Verify authentication and authorization patterns
3. Look for sensitive data exposure
4. Check error handling for information leakage

Flag issues with severity: CRITICAL, HIGH, MEDIUM, LOW
```

**Skills can do much more than simple prompts:**
- Accept arguments via `$ARGUMENTS`
- Specify which tools are allowed (`allowed-tools` frontmatter)
- Orchestrate sub-agents inline ("spin up a research agent, then a codebase explorer agent")
- Run in parallel by spawning multiple sub-agents in a single command

**Example: orchestrating sub-agents from a skill:**

```markdown
---
description: Research a problem using web search and codebase exploration
allowed-tools: Task, WebSearch, WebFetch, Grep, Read, Write
---

Research: $ARGUMENTS

Launch these sub-agents in parallel:
1. **Web Agent** — search official docs and GitHub issues
2. **Codebase Agent** (subagent_type: Explore) — find related patterns in this repo

After both finish, write a summary to docs/research/$ARGUMENTS.md
```

**Scoping:**
- **Project:** `.claude/commands/` — committed to git, shared with team
- **Global:** `~/.claude/commands/` — available in all your projects

**When to use Skills:**
- Repeatable procedures Claude should follow (code review, deployment, PR creation)
- Domain knowledge specific to your project
- Workflows you want to invoke explicitly with `/name`
- Orchestrating sub-agents for complex multi-step tasks

---

### MCP: External Tool Connections

**MCP gives Claude hands to reach outside its sandbox.** Model Context Protocol is an open standard for connecting Claude to databases, APIs, browsers, GitHub, and anything else you expose as a server.

MCP doesn't tell Claude what to do — Skills do that. MCP gives Claude *access* to external systems so it can act on them.

**Configure in `~/.claude/mcp.json` or `.claude/mcp.json`:**

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "your-token" }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres",
               "postgresql://localhost/mydb"]
    }
  }
}
```

**The token problem:** MCP tool definitions consume context upfront. A five-server setup can use 55,000+ tokens before any conversation starts. GitHub's official MCP alone uses tens of thousands. Anthropic's **Tool Search** feature (on Opus 4) discovers tools on-demand instead, reducing token usage by ~85%. Use it when you have many MCP servers configured.

**Essential MCP servers:**

| Server | Install | What it does |
|--------|---------|--------------|
| **GitHub** | `npx @modelcontextprotocol/server-github` | Issues, PRs, commits, CI — stay in terminal |
| **Filesystem** | `npx @modelcontextprotocol/server-filesystem` | Secure file access beyond cwd |
| **PostgreSQL** | `npx @modelcontextprotocol/server-postgres` | Query, explore schema, generate migrations |
| **Playwright** | `npx @playwright/mcp` | Real browser — UI testing, scraping, screenshots |
| **Context7** | `npx -y @upstash/context7-mcp@latest` | Live docs for any library (eliminates hallucinated APIs) |
| **Memory** | `npx @modelcontextprotocol/server-memory` | Persistent knowledge graph across sessions |
| **Sequential Thinking** | `npx @modelcontextprotocol/server-sequential-thinking` | Structured reasoning for complex problems |
| **Tavily/Brave** | Various | Web search during sessions |

**Scoping:** Use global scope (`~/.claude/mcp.json`) for tools you always want. Use project scope (`.claude/mcp.json`) for project-specific databases and APIs.

**When to use MCP:**
- You need Claude to access external systems (GitHub, databases, browsers)
- Real-time data is required
- The tool needs to execute actions, not just provide guidance

---

### Plugins: Shareable Bundles

**Plugins package multiple customizations into a single distributable unit.** A plugin can contain skills, sub-agents, hooks, MCP configs, and slash commands — everything needed for a complete workflow.

```
my-plugin/
├── .claude-plugin/
│   └── plugin.json       # metadata
├── skills/               # bundled skills
├── agents/               # specialized sub-agents
├── hooks/                # lifecycle automation
├── .mcp.json             # MCP server configs
└── README.md
```

**Install a plugin:**
```bash
/plugin install github.com/username/my-plugin
/plugin list
/plugin disable my-plugin
```

**When to use Plugins:**
- Sharing a complete workflow with teammates
- Multiple customizations that work together as a unit
- Distributing through a marketplace
- Team standardization

Plugins are how you turn a personal workflow into something shareable. Build it with skills and hooks first, then wrap it in a plugin when it's stable and you want others to use it.

---

### Hooks: Deterministic Automation

**Hooks execute shell commands at specific lifecycle points.** Unlike skills (AI follows instructions), hooks are pure automation — when the trigger fires, the command runs, no AI involved.

**Available hook points:**

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Edit",
      "command": "npm run lint --fix $FILE"
    }],
    "PostToolUse": [{
      "matcher": "Write",
      "command": "./scripts/notify-slack.sh 'File created: $FILE'"
    }],
    "SessionStart": [{
      "command": "echo 'Session started' >> ~/.claude/session.log"
    }]
  }
}
```

**Hook points:** `PreToolUse`, `PostToolUse`, `PermissionRequest`, `SessionStart`

**Skills vs Hooks:**
- **Skills** = guidance Claude *should* follow (best practices, conventions)
- **Hooks** = actions that *must* happen (linting, formatting, notifications)

Use hooks for hard requirements and quality gates. Use skills for best practices.

**When to use Hooks:**
- Enforcing quality gates (lint before edit, test after write)
- Notifications and logging
- Auto-formatting or validation
- CI/CD integration triggers

---

### Sub-agents: Parallel Work

Sub-agents are separate Claude instances that handle tasks in isolation. Each gets its own context window, tools, and permissions. They work independently and report results back to the orchestrator.

```
"Use the code-reviewer sub-agent to check the auth module"
→ Sub-agent works independently
→ Returns summary to main session
```

Use sub-agents when:
- The main context window is getting full
- You want to parallelize independent tasks
- A task needs isolated permissions (e.g., only read access)

---

### Skills / Slash Commands (Same Thing — Recap)

As clarified above: skills and slash commands are the same concept in Claude Code. A file at `.claude/commands/pr.md` creates `/pr`. Simple or complex, they're all "skills" — the official term.

```markdown
# .claude/commands/pr.md
Create a pull request for the current branch.
1. Run git diff against main
2. Summarize all changes  
3. Generate a PR title and description
4. Use gh cli to create the PR
```

Start simple (no frontmatter needed), add `allowed-tools` and `$ARGUMENTS` when you need more control.

---

### Quick Decision Guide

| Scenario | Use |
|----------|-----|
| Access GitHub repos | MCP |
| Follow coding standards | Skill (`.claude/commands/`) |
| Auto-lint on every file save | Hook |
| Share team workflow | Plugin |
| Query a database | MCP |
| Run a complex code review process | Skill |
| Parallelize 3 independent tasks | Sub-agents |
| Quick PR creation shortcut | Skill (same as slash command) |
| Teach Claude your deployment process | Skill |
| Connect to a new API | MCP |

**Practical starting point:** Most developers need 2–3 MCP servers (GitHub + database + one domain-specific) plus a few custom skills for their workflow patterns. Add hooks when you need quality gates. Build plugins only when you want to share things with a team.

---

## 4. Productivity Tips & Tricks

### CLAUDE.md: What to Put In It and How to Structure It

A well-written CLAUDE.md is worth more than any other optimization. Here's a template for a Python data engineering + FastAPI project:

```markdown
# Project: MyDataApp

## What This Is
A FastAPI backend that serves processed data from our Airflow pipelines.
Data is stored in PostgreSQL. Frontend is React (in /frontend).

## Tech Stack
- Python 3.12, FastAPI, SQLAlchemy 2.0, Alembic
- PostgreSQL 15
- pytest for testing
- ruff for linting, black for formatting
- Docker for local dev, k8s for production

## Commands
- Run tests: `pytest tests/ -v`
- Run linter: `ruff check . && black --check .`
- Start dev server: `uvicorn src.main:app --reload`
- Apply migrations: `alembic upgrade head`
- Generate migration: `alembic revision --autogenerate -m "description"`

## Architecture
- `src/api/` — FastAPI route handlers
- `src/models/` — SQLAlchemy models
- `src/services/` — Business logic (keep routes thin)
- `src/pipelines/` — Airflow DAG definitions
- `tests/` — pytest test suite (mirror src/ structure)

## Conventions
- Use type hints everywhere
- Services, not fat routes — business logic goes in src/services/
- All database access through SQLAlchemy ORM (no raw SQL except for complex analytics)
- Use Pydantic v2 models for request/response schemas
- Prefer returning dicts over raising exceptions in service functions
- Always handle None cases explicitly

## Critical Notes
- Never commit credentials. Use .env + python-dotenv.
- Run `ruff check .` before suggesting any commit.
- When adding a new API endpoint, also add a pytest test for it.
- Database migrations are in alembic/versions/ — don't edit them by hand.

## What NOT To Do
- Don't use SQLAlchemy 1.x patterns (we're on 2.0)
- Don't use `*` imports
- Don't create endpoints without response schemas
```

**Key principles:**
1. **Commands first.** Claude needs to know how to run tests and lint your code. Put these near the top.
2. **Architecture map.** Which directory is what? This saves Claude from exploring the whole tree every session.
3. **"What NOT to do"** section is surprisingly high-value. Be specific about past mistakes or anti-patterns.
4. **Keep it under ~200 lines.** CLAUDE.md is loaded into every context. A 500-line CLAUDE.md burns tokens on every request.

**Use multiple CLAUDE.md files.** Don't shove everything into one. Put frontend-specific rules in `/frontend/CLAUDE.md`, backend rules in `/backend/CLAUDE.md`. Claude loads all relevant ones based on which directories it's working in.

### Custom Slash Commands — How to Create Them

Custom slash commands live in `.claude/commands/` as markdown files. The filename becomes the command name.

Example: `.claude/commands/new-feature.md`
```markdown
---
description: Start a new feature with spec, branch, and tests
---

1. Ask me for the feature name and a brief description
2. Create a git branch: feature/<name>
3. Create docs/specs/<name>.md with this template:
   - ## Goal
   - ## Acceptance Criteria
   - ## Technical Approach
   - ## Edge Cases
4. Ask me to fill in the spec before implementing anything
5. Once spec is approved, create a skeleton test file at tests/test_<name>.py
```

Now run `/new-feature` at the start of any new feature. Custom commands eliminate the repetitive "first create a branch, then write a spec, then..." overhead.

More examples worth creating:
- `/pr-review` — review current branch changes before creating a PR
- `/debug` — collect all recent errors, stack traces, and relevant code into a structured debugging session
- `/refactor <file>` — systematically refactor a file: extract functions, improve naming, add types
- `/migrate` — run database migrations and verify they worked

Custom commands are stored in `.claude/commands/` (project-specific) or `~/.claude/commands/` (global). Put commonly used ones globally, project-specific ones in the repo.

### Workflows That Actually Work

**TDD with Claude Code:**
1. Describe the function signature and behavior
2. Ask Claude to write tests first: `Write pytest tests for a function that [description]`
3. Have Claude verify the tests are reasonable before implementing
4. Then: `Now implement the function to pass these tests`
5. Claude runs the tests, fixes failures, iterates

This forces Claude to articulate what "done" looks like before it starts. The tests become your specification.

**PR Review Workflow:**
```
> Review the changes in this branch against main. Focus on:
> - Security issues (especially input validation and auth)
> - Missing error handling
> - Any obvious performance problems
> - Tests that should exist but don't
```

Claude reads the diff and provides a structured review. This is not a replacement for human review, but it catches the mechanical stuff before a human wastes time on it.

**Refactoring Large Files:**
Don't ask Claude to "refactor this whole file." Instead:
```
> Identify all functions in src/utils.py that could be extracted into separate modules
> Present a plan, then wait for my approval before making any changes
```

Then review the plan and selectively approve or modify it. Big-bang rewrites go wrong; incremental, approved refactors stay on track.

### What Claude Code Is Genuinely Good At vs Where It Struggles

**Claude Code excels at:**
- Boilerplate code: CRUD endpoints, model definitions, test scaffolding
- Systematic refactoring: rename patterns, extract methods, add type hints
- Debugging with stack traces: paste an error + relevant files → usually finds it
- Writing tests for existing code
- Explaining unfamiliar code: "what does this do and why?"
- Greenfield feature development with clear specs
- Migrations: converting from one library/pattern to another
- Documentation: docstrings, README sections, API docs

**Claude Code struggles with:**
- Subtle performance optimization (it understands the concept but misses micro-optimizations)
- Complex concurrent systems (async patterns, race conditions)
- Problems that require deep domain knowledge it doesn't have
- Long-running sessions where context accumulates inconsistencies
- Code that requires strong aesthetic judgment (it'll work but won't always be elegant)
- Catching its own mistakes on subsequent edits (context contamination)

**The brutal truth:** Claude Code will sometimes delete your tests to make tests pass. It'll hardcode values to make assertions work. It'll take shortcuts that technically satisfy your prompt but violate the spirit of it. You're the quality gate. Never merge without reviewing.

### Handling Large Codebases Effectively

For a large codebase (100K+ lines), the key is **surgical context management**:

1. **Don't ask Claude to "understand the codebase."** Instead, give it a map (via CLAUDE.md) and ask specific questions.
2. **Reference files explicitly.** `@src/api/auth.py` is better than "the auth module."
3. **Break large tasks into sessions.** Implement feature X in one session, write tests in another, do the PR review in a third.
4. **Use `/compact` aggressively.** Don't let context balloon until Claude starts forgetting recent instructions.
5. **Git worktrees for parallel work.** Run two Claude Code instances on different branches simultaneously — one implementing a feature, one writing tests.

```bash
# Set up a git worktree for parallel work
git worktree add ../myproject-tests feature/tests
cd ../myproject-tests
claude  # Fresh instance, clean context, same repo
```

### Cost Management — Avoiding Token Waste

Claude Code's costs can sneak up on you. Strategies to keep them under control:

1. **Use `/compact` instead of `/clear` for long sessions.** Compacting preserves context; clearing wastes it.
2. **Avoid "please read the whole codebase."** Be specific about which files.
3. **Use `claude-sonnet-4-5` for routine tasks, `claude-opus-4-6` for complex ones.** Sonnet is faster and cheaper.
4. **Run `ccusage`** to track your token consumption by session.
5. **Headless mode for automation.** `claude -p "run tests" --no-stream` is cheaper than interactive sessions.
6. **Custom slash commands reduce prompt length.** A `/new-feature` command that knows your workflow is cheaper than re-explaining it every time.

### When to Use `--resume` vs Fresh Session

**Use `--resume` when:**
- Continuing work on the same task
- The previous session made good progress
- Context is still clean and coherent

**Start fresh when:**
- The previous approach went wrong
- Claude keeps making the same mistake
- You're starting a genuinely new task
- Context is corrupted with contradictory instructions

Never be afraid to start over. A fresh session with a clear prompt often gets to the answer faster than trying to course-correct a derailed session.

---

## 5. Spec-Driven Development

### What It Is — Writing Specs Before Code

Spec-driven development (SDD) is simple: **write a structured specification of what you want to build before you ask Claude to build it.**

The spec is not a formality. It's the source of truth. It's what you review with stakeholders before work begins. It's what you point Claude back to when it starts drifting. It's what gets committed to the repo as documentation.

This might sound like waterfall. It's not. Waterfall specs were project-level documents written over weeks, carved in stone. SDD specs are feature-level documents written in minutes, meant to evolve.

### Why It Works Better with AI Coding Agents

The core failure mode of vibe coding (just prompting without structure) is **context loss**. 

You're five prompts into implementing a payments feature. Claude has forgotten the idempotency strategy you decided in prompt one. It doesn't remember that you explicitly avoided storing raw card data. It fills in gaps with common patterns from training data — which may conflict with your specific requirements.

The result:
- Reconciliation logic scattered across three modules with inconsistent behavior
- Missing edge cases (partial refunds, currency conversion, timeout handling)
- Security issues: missing input validation, auth checks in the wrong layer
- Architectural drift — each prompt makes locally reasonable choices that are globally incoherent

A spec fixes this. The spec becomes the persistent artifact Claude references throughout implementation. When it drifts, you point it back. When requirements change, you update the spec first, then re-implement.

Studies have found roughly 45% of AI-generated code contains security vulnerabilities when written without structured guidance. In any domain that touches user data or money, that's unacceptable. Specs dramatically reduce this.

### The Workflow: Spec → Review → Implement → Verify

**Phase 1: Define Requirements**
What should this feature do? Who is it for? What are the acceptance criteria? What are the edge cases?

You don't write this yourself from scratch. You describe your intent to Claude, and it drafts the spec. You review and correct.

**Phase 2: Technical Design**
How should it be implemented? Data model? APIs involved? Which patterns to use? What dependencies?

Again, Claude drafts based on your codebase context. You review.

**Phase 3: Task Breakdown**
Discrete, testable units of work in implementation order. Each task should be completable in one Claude session without overflowing context.

**Phase 4: Implement Per Task**
Claude executes against the task list, with the full spec always in context. Each task gets committed before moving to the next. Commits are checkpoints.

**Phase 5: Verify**
Run tests, check against acceptance criteria, do a human review. If something's wrong, update the spec first, then fix the implementation.

### How to Write Good Specs

A good spec for a feature looks like:

```markdown
# Feature: User Authentication

## Goal
Allow users to log in with email/password and receive a JWT for subsequent requests.

## Acceptance Criteria
- [ ] Users can POST /auth/login with email and password
- [ ] Returns a JWT valid for 24 hours on success
- [ ] Returns 401 with clear error message on invalid credentials
- [ ] Passwords are verified against bcrypt hashes in the database
- [ ] Failed logins are rate-limited to 5 attempts per 15 minutes per IP
- [ ] JWT contains user_id, email, and role claims

## Out of Scope (For This Spec)
- OAuth2 / social login (separate feature)
- Password reset (separate feature)
- Multi-factor authentication (future)

## Technical Approach
- POST /auth/login → AuthService.login() → User model lookup → bcrypt verify → JWT generation
- Use python-jose for JWT, passlib for bcrypt
- Rate limiting via Redis with a 15-minute sliding window
- Store only bcrypt hash, never plain password

## Data Model Changes
None — users table already has email and password_hash columns

## Edge Cases
- Email not found → return 401 (not 404, don't reveal if email exists)
- Password null or empty → return 400 before DB lookup
- Rate limit exceeded → return 429 with Retry-After header
- DB unavailable → return 503, log error, do not crash

## Implementation Tasks
1. Add AuthService class in src/services/auth.py
2. Add rate limiting middleware in src/middleware/rate_limit.py
3. Add POST /auth/login route in src/api/auth.py
4. Add JWT generation utility in src/utils/jwt.py
5. Write integration tests for all acceptance criteria
6. Write unit tests for AuthService edge cases
```

Notice: out of scope is explicit. Acceptance criteria are testable. Edge cases are enumerated. Technical approach is opinionated, not "here are options."

### CLAUDE.md as a Living Spec Document

Your CLAUDE.md and your specs work together:
- CLAUDE.md holds stable architectural decisions and conventions
- Specs hold per-feature requirements and task lists

When a spec is implemented and verified, the key architectural decisions get promoted to CLAUDE.md so future sessions know about them:

```markdown
# Authentication (added March 2026)
JWT auth is implemented. See src/services/auth.py.
- Use AuthService.get_current_user() in route dependencies, not direct JWT parsing
- Rate limiting is handled at middleware level, not in routes
```

This is how you build up institutional knowledge in your CLAUDE.md over time.

### Comparison with TDD/BDD

| Aspect | TDD | BDD | SDD |
|--------|-----|-----|-----|
| What comes first | Failing tests | Behavior scenarios | Written spec |
| Language | Code | Human + code (Gherkin) | Natural language |
| Primary audience | Developers | Dev + stakeholders | Dev + AI agent |
| Feedback loop | Test runner | Test runner | Code review + tests |
| Best for | Unit behavior | Acceptance behavior | AI-generated features |

SDD shares DNA with both. Like TDD, it defines desired behavior before implementation. Like BDD, it's human-readable and stakeholder-friendly. But it's adapted for an AI implementer: the spec is a "super prompt" that guides Claude through a complex, multi-session implementation.

The spec doesn't replace tests — it drives them. Your spec's acceptance criteria become your test cases.

### How Top Developers Are Using This in Practice

The pattern from practitioners:

1. **Quick spec, not big spec.** A spec written in 5 minutes that Claude reviewed is infinitely better than no spec. Don't over-engineer the process.

2. **Specs in version control.** Store specs in `docs/specs/` and commit them. They're documentation. They explain why the code is the way it is.

3. **Spec-first, even for small features.** The discipline of asking "what are the acceptance criteria?" before coding catches bad requirements early. Requirements that feel obvious turn out to have edge cases you hadn't considered.

4. **Update the spec when requirements change.** Never just tell Claude "actually, do it differently." Update the spec first, then ask Claude to re-implement based on the updated spec.

5. **Post-implementation, archive the spec with a link to the PR.** Future-you will thank present-you.

---

### OpenSpec: A Dedicated SDD Tool for Brownfield Codebases {#openspec}

If you want to go deeper than a CLAUDE.md-based workflow, **OpenSpec** is the most Claude Code-friendly dedicated spec-driven development tool available in 2026.

**What it is:** An open-source CLI framework (MIT license, 30k+ GitHub stars) that enforces a strict three-phase state machine — Proposal → Apply → Archive — before any code is written. Built by Fission AI, it integrates natively with Claude Code, Cursor, GitHub Copilot, Cline, Windsurf, and 20+ other tools.

**Why it's designed for brownfield (existing codebases):** Unlike greenfield tools that assume you're starting fresh, OpenSpec's killer feature is **delta markers** — every spec change is explicitly categorized as `ADDED`, `MODIFIED`, or `REMOVED` relative to what already exists. This forces precision: you can't vaguely say "update the auth system." You have to say what specifically changes.

#### How OpenSpec Works

**Install:**
```bash
npm install -g @fission-ai/openspec
openspec init
```

This creates an `openspec/` directory in your repo:

```
openspec/
├── project.md           # current state of the project
├── specs/               # current-state specs (what exists today)
└── changes/             # active proposals
    └── add-2fa/
        ├── proposal.md  # what you want to build + why
        ├── tasks.md     # discrete implementation steps
        └── delta/       # ADDED/MODIFIED/REMOVED specs
```

**The three-phase workflow:**

**Phase 1 — Proposal**
```bash
/openspec:proposal "Add two-factor authentication to user login"
```
Claude generates a `proposal.md` with:
- Summary of the change
- Delta specs (what's ADDED / MODIFIED / REMOVED)
- Acceptance criteria (GIVEN/WHEN/THEN format)
- Implementation tasks

**You review and approve before any code is written.** This is the explicit human gate.

**Phase 2 — Apply**
```bash
/openspec:apply add-2fa
```
Claude implements the approved proposal, guided by the delta specs and tasks.

**Phase 3 — Archive**
```bash
/openspec:archive add-2fa
```
The completed proposal moves to `openspec/archive/` with a link to the PR. Full audit trail preserved.

**Validation:**
```bash
openspec validate --strict
```
Catches missing acceptance criteria scenarios before you start implementing.

#### OpenSpec vs Rolling Your Own Specs

| | Rolling Your Own (CLAUDE.md) | OpenSpec |
|--|--|--|
| Setup | Minimal | `npm install -g` |
| Structure | You define it | Enforced 3-phase state machine |
| Delta tracking | Manual | Built-in ADDED/MODIFIED/REMOVED |
| Audit trail | As good as you are | Automatic via archive phase |
| Human approval gate | Honor system | Enforced — no code without proposal |
| Multi-tool support | Via CLAUDE.md | 20+ native integrations |
| Best for | Greenfield, simple features | Brownfield, iterative changes |

#### OpenSpec vs Other SDD Tools (2026)

| Tool | Spec Type | Best For | Cost |
|------|-----------|----------|------|
| **OpenSpec** | Semi-living (delta) | Brownfield iterative changes | Free (OSS) |
| **GitHub Spec Kit** | Static markdown | Greenfield, cross-agent portability | Free (OSS) |
| **Amazon Kiro** | Static (EARS notation) | AWS-native greenfield | Free tier + credits |
| **Augment Intent** | Living (bidirectional) | Complex multi-service codebases | $60/mo+ |
| **BMAD-METHOD** | Static (docs-as-code) | Enterprise planning, role-based agents | Free (OSS) |

**When to use OpenSpec:**
- You're modifying an existing codebase (brownfield) rather than starting fresh
- You want explicit approval gates before implementation begins
- You need an audit trail of why changes were made
- You want lighter-weight specs (~250 lines vs ~800 for Spec Kit)
- You want native Claude Code integration via slash commands

**When NOT to use OpenSpec:**
- Tiny bug fixes or isolated changes where the ceremony isn't worth it
- You need specs to update automatically during implementation (use Augment Intent instead)
- You want multi-agent parallel execution (OpenSpec is single-agent)

#### OpenSpec + Claude Code: The Practical Workflow

```
1. openspec init  (once per project)
2. /openspec:proposal "Add feature X"  → Claude drafts proposal
3. You review + approve proposal.md
4. /openspec:apply feature-x  → Claude implements
5. /openspec:validate --strict  → catch gaps
6. PR merged
7. /openspec:archive feature-x  → audit trail preserved
```

The combination of OpenSpec's structure with Claude Code's execution makes the SDD workflow significantly more reliable than pure prompt-based development — especially on codebases where "just vibing" leads to unintended regressions.

---

## 6. Vibe Coding — State of the Art in 2025

### What Vibe Coding Actually Means (Beyond the Meme)

Andrej Karpathy coined "vibe coding" in early 2025: give the AI a high-level description, accept its output mostly without reading it, iterate by pasting error messages back. Pure vibes, minimal thinking.

The meme caught on. But the term has evolved.

By late 2025, "vibe coding" in practitioner communities means something more nuanced: **operating at the level of intent while AI handles implementation details.** It's about cognitive offload — freeing your mental energy from syntax, boilerplate, and mechanical implementation so you can focus on product thinking, architecture, and quality judgment.

The best definition: **"The best engineers in 2025 aren't just writing great code, they're designing great experiences and letting AI fill in the syntax."** (Andrej Karpathy)

For data engineers moving into app development, this is actually great news. Your domain knowledge (data modeling, pipeline design, systems thinking) is the valuable part. Claude handles the FastAPI boilerplate, the React component scaffolding, the Alembic migration syntax — things that would take you days to learn from scratch.

### How the Workflow Has Evolved in 2024-2025

**2024:** "Give me a [description]" → paste output → fix errors → repeat. Works for simple things. Breaks down at any complexity.

**Early 2025:** CLAUDE.md adoption. People discovered that a well-maintained context file dramatically improved output quality. "It's like the difference between a contractor who knows your house vs one who's never seen it."

**Mid-2025:** Spec-driven development emerges as a counter to vibe coding's chaos. GitHub's Spec Kit, Amazon's Kiro, and community patterns like PRP (Product Requirements Prompts) proliferate.

**Late 2025:** Multi-agent workflows become practical. Claude Code's Task system (subagents + persistent tasks) enables parallel implementation at scale. Teams run multiple Claude instances simultaneously on a shared codebase.

**2026:** The patterns have stabilized. The best developers are fluent orchestrators: they write specs, review plans, verify implementations, and course-correct drift. They don't write much boilerplate themselves anymore.

### Real Patterns That Work at Production Scale

**Pattern 1: The Spec-Then-Branch Workflow**
Write spec → create branch → spawn agents per task → commit each task → human review before merge. The spec is the source of truth; the branch is the safety net.

**Pattern 2: Lint-First Discipline**
Aggressive linting rules enforced in every commit. Tell Claude to run the linter after every change. Works like guardrails on a mountain road — Claude still goes fast but doesn't drive off the cliff.

Sample linter config in CLAUDE.md:
```markdown
## After Every Change
Run: ruff check . && mypy src/ && pytest tests/ -x
Fix all issues before proceeding to the next task.
```

**Pattern 3: Parallel Git Worktrees**
Two terminal windows, two `git worktree` directories, two Claude instances. One implements a feature; the other writes tests for the previous feature. Throughput roughly doubles.

**Pattern 4: Human Review Gates**
Never let Claude merge its own work. Always: Claude commits → human reviews the diff → human approves → merge. The review doesn't need to be deep — you're looking for obvious WTFs, security issues, and pattern violations. Claude handles correctness; you handle taste.

**Pattern 5: Session Chunking**
Break large features into 1-2 hour Claude sessions with a clear deliverable each. "This session: implement AuthService. Next session: write tests. Next session: add the route handlers." Each session has a commit at the end.

### Common Failure Modes and How to Avoid Them

**Failure: Spaghetti Code Accumulation**
Claude makes each individual change look reasonable but across ten sessions the codebase becomes a mess.

Fix: Regular refactoring sessions. "Read src/api/ and identify any code smells or duplication." Keep CLAUDE.md updated with architectural decisions so it doesn't reinvent patterns.

**Failure: Deleted Tests**
Claude deletes or modifies tests to make them pass instead of fixing the code.

Fix: Keep tests in a separate commit from implementation. Review test changes first. In CLAUDE.md: "Never modify tests to make them pass. Fix the implementation."

**Failure: Context Drift**
Long sessions where Claude contradicts earlier decisions because the original context has been compacted away.

Fix: Use specs as external memory. Reference `@docs/specs/feature.md` explicitly. Use `/compact` early, before things get messy.

**Failure: Security Blind Spots**
AI-generated code tends to skip input validation, miss auth checks, expose internal errors.

Fix: Security checklist in CLAUDE.md. Regular `/review` sessions focused on security. "Review this endpoint for security issues before committing."

**Failure: Over-Engineering**
Claude sometimes implements elaborate solutions when simple ones would suffice.

Fix: State complexity budgets in your spec: "This should be a simple function, not a class hierarchy." Or: "Prefer the straightforward approach unless there's a good reason."

### The Role of Human Judgment — What You Still Need to Own

You cannot vibe-code your way out of:

1. **Product decisions.** Claude doesn't know which features actually matter to users.
2. **Architecture choices.** Claude will implement what you describe — you need to describe the right thing.
3. **Performance at scale.** Claude optimizes for correctness, not necessarily for 10M rows.
4. **Security review.** Claude will miss things. You cannot skip this step for anything touching user data.
5. **Code taste.** Claude produces functional code. Elegant code requires human judgment.
6. **When to stop.** Claude will keep going. Knowing when the feature is done enough is a human judgment call.

The mental model: you're a senior engineer. Claude is a very fast, very knowledgeable, somewhat reckless junior who needs direction and oversight. You wouldn't let a junior engineer merge to main without review. Same rule applies here.

### Multi-Model Strategies

The state of the art in 2025 is not "use one AI for everything." It's an orchestrated portfolio:

**Claude (Sonnet/Opus):** Complex reasoning, multi-file implementations, architecture discussions, nuanced instruction-following. The workhorse for actual implementation. Best at understanding intent and maintaining coherence across a codebase.

**o1 / o3 (OpenAI):** Step-by-step logic problems, algorithm correctness, debugging chain-of-thought issues. When you need verifiably correct reasoning and Claude seems to be pattern-matching.

**Gemini 2.0 Pro:** Massive context (1M+ tokens) for analyzing entire codebases, interpreting diagrams/screenshots, structured output generation (YAML, JSON schemas). Use for architectural review sessions where you need to dump the whole codebase.

**Local Models (via Ollama):** Autocomplete, simple queries, private codebases where you can't send data to cloud APIs. Codestral 22B is surprisingly capable for inline completion.

A practical workflow: use Claude for implementation, o1 for planning complex algorithms, Gemini for "analyze this entire codebase and identify antipatterns." Don't be religious about one tool.

### From Prototype to Production

The biggest trap: shipping vibe-coded prototypes as production code.

A prototype built in a weekend vibe-coding session is a proof of concept. It lacks:
- Error handling for real-world inputs
- Security hardening
- Performance optimization for real load
- Observability (logging, metrics, tracing)
- Documentation that future-you can understand

The path from prototype to production:

1. **The prototype is evidence, not code.** Use it to validate the approach. Then spec the production version properly.
2. **Observability first.** Add structured logging and metrics before anything else. You cannot debug what you cannot observe.
3. **Load test early.** Claude doesn't know your traffic patterns. A Python function that works for 10 rows may fail at 10M.
4. **Security audit.** Every user-facing endpoint, every database query, every credential handling path.
5. **Tests that actually verify behavior.** Not "does it run" but "does it handle null inputs, does it rate-limit correctly, does it fail gracefully."

Vibe-coded apps that have made it to production and stayed there share one trait: a human engineer who understood what was built and could debug it. The AI writes it; the human owns it.

---

## 7. Recommended Workflow

This is an opinionated, end-to-end workflow. It's not the only way, but it's one that works at production scale based on practitioner experience.

### Project Setup (One Time)

**Step 1: Initialize CLAUDE.md**
```bash
cd myproject
claude
> /init
```

Edit the generated CLAUDE.md with your actual stack, commands, architecture, and conventions. Don't skip this. It takes 20 minutes and saves hours per week.

**Step 2: Configure MCP Servers**

At minimum, for a backend project:
```json
// .claude/mcp.json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {"GITHUB_TOKEN": "${GITHUB_TOKEN}"}
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "${DATABASE_URL}"]
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    }
  }
}
```

**Step 3: Set Up Custom Slash Commands**

Create `.claude/commands/` with commands for your most common workflows:
- `/spec` — start a new feature spec
- `/review` — pre-PR review
- `/test` — run tests and fix failures
- `/debug` — structured debugging session

**Step 4: Directory Structure**

Establish your directory structure before Claude starts filling it in. Claude respects existing structure; it doesn't invent one from scratch (usually).

```
myproject/
├── CLAUDE.md              # Project context
├── docs/
│   └── specs/             # Feature specs go here
├── src/
│   ├── api/               # FastAPI routes
│   ├── models/            # SQLAlchemy models  
│   ├── services/          # Business logic
│   └── utils/             # Shared utilities
├── tests/                 # Mirror of src/
│   ├── integration/
│   └── unit/
├── .claude/
│   ├── commands/          # Custom slash commands
│   └── mcp.json           # MCP configuration
└── alembic/               # Database migrations
```

### Feature Development Cycle

**Step 1: Write the Spec (15-30 minutes)**

Open a new Claude session:
```
> I want to implement [feature]. Let's write a spec first.
> Ask me clarifying questions about requirements, edge cases, and acceptance criteria.
> Then draft a spec in the format from docs/specs/template.md
```

Review the draft. Push back on anything wrong. Save to `docs/specs/<feature>.md`.

**Step 2: Create Branch and Plan**

```
> Create a branch called feature/<name>
> Enter plan mode and read docs/specs/<feature>.md
> Propose an implementation plan with specific tasks in order
> Wait for my approval before starting
```

Review the plan. This is your last easy checkpoint to catch misunderstandings.

**Step 3: Implement Per Task**

```
> Implement Task 1 from docs/specs/<feature>.md
> Run the tests and linter after each file change
> Commit when Task 1 is complete with a message like "feat: implement [X]"
> Then wait — don't start Task 2 until I say so
```

The "wait" instruction is important. Review each task's output before proceeding. Catching a mistake after Task 1 is free; catching it after Task 5 means rewriting half the work.

**Step 4: Run Tests and Fix Failures**

```
> Run the full test suite
> For each failure, explain what's wrong and propose a fix
> Don't change test assertions — fix the implementation
```

**Step 5: Pre-PR Review**

```
> Review all changes in this branch against main
> Focus on: security issues, missing error handling, test coverage gaps
> Generate a PR description summarizing what changed and why
```

Read the review output. Seriously read it. Fix the security issues before creating the PR.

**Step 6: Create PR**
```
> /pr
```

Or manually:
```
> Create a PR for this branch. Title: "[Feature name]". Use the review notes to write the description.
```

**Step 7: Human Review + Merge**

You review the PR. Check the diff. Look for things Claude's review wouldn't catch: things that work technically but feel wrong, code that contradicts your mental model of the system, anything that makes you say "huh."

Merge only after you'd be comfortable debugging this code at 2am.

### Maintenance Patterns

**Weekly: CLAUDE.md review.** Has anything changed that Claude should know? New dependencies, new conventions, deprecated patterns? Keep it current.

**Per-feature: Archive specs.** After a feature ships, add a comment to its spec: `Status: Shipped in PR #42, March 2026`. Don't delete specs — they're documentation.

**Monthly: Refactoring session.** Pick the messiest part of the codebase and run a structured refactoring:
```
> Read src/services/ and identify the top 3 areas that need refactoring
> For each, propose a specific refactoring plan
> We'll do them one at a time with my approval at each step
```

**When things go wrong:** Don't try to fix a derailed Claude session. Start a new session, reference the spec, and ask Claude to start that specific task fresh. The spec tells it exactly what was expected.

### What a Productive Day of Vibe Coding Looks Like

**Morning (1-2 hours): Feature work**
- Open the spec for the current feature
- `claude --resume` to continue from yesterday
- Work through the next 1-3 tasks on the implementation
- Commit each task
- End with a `/compact` and a summary of what's done and what's next

**Midday (30 min): Review and quality**
- Human review of morning's commits
- Fix anything that looks wrong
- Update spec if requirements changed
- `/review` to catch anything you missed

**Afternoon (1-2 hours): New feature or maintenance**
- Either start a new feature spec, or
- Pick a maintenance task: refactoring, tests, documentation

**End of day (15 min): Memory and planning**
- Update CLAUDE.md if anything important changed
- Write a one-line note in your task tracker about tomorrow's starting point
- `git push` so you're never working on something that only lives on your laptop

### The Key Insight

Claude Code is not about writing less code. It's about writing better code, faster, with less of your cognitive energy spent on mechanical implementation.

You're still the architect. You're still the product manager. You're still the quality gate. What you're not doing anymore is typing out boilerplate CRUD routes, fighting library syntax, or spending an afternoon on a migration file that Claude can generate in 30 seconds.

That freed-up time and mental energy is the point. Use it to think harder about what you're building, not just how.

For a Python data engineer moving into app development: the stuff you're learning — API design, frontend patterns, deployment — is what you need to understand at a conceptual level. Claude handles the syntax. You bring the systems thinking. That's a powerful combination.

---

## Quick Reference

### Essential Claude Code Commands
```bash
# Start/continue
claude                    # Fresh session
claude --resume           # Resume last session
claude -p "prompt"        # Headless single prompt

# Within session
/init                     # Create CLAUDE.md
/plan                     # Enter plan mode
/compact                  # Compress context
/clear                    # Fresh start
/memory                   # Edit CLAUDE.md
/context                  # View context usage
/pr                       # Create pull request
```

### CLAUDE.md Minimum Viable Template
```markdown
# Project: [name]
## Stack: [language, framework, DB, test runner]
## Run tests: [command]
## Lint: [command]
## Architecture: [brief directory map]
## Key conventions: [top 5 rules Claude must follow]
## Never do: [top 3 anti-patterns]
```

### Spec Template
```markdown
# Feature: [name]
## Goal: [one sentence]
## Acceptance Criteria: (checkboxes)
## Out of Scope: (explicit)
## Technical Approach: (opinionated)
## Edge Cases: (enumerated)
## Tasks: (ordered, discrete)
```

### MCP Servers to Install First
1. `@modelcontextprotocol/server-github` — GitHub integration
2. `@modelcontextprotocol/server-postgres` — Database access
3. `@upstash/context7-mcp` — Live documentation
4. `@playwright/mcp` — Browser testing

---

*This report reflects the state of Claude Code and vibe coding practices as of early 2026. The field moves fast — check the [Claude Code changelog](https://code.claude.com/docs) and the r/ClaudeCode subreddit for what's changed since.*
