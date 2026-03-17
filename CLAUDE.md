# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

This repo is a collection of shared AI assistant rules, skills, and agents for Claude Code (and mirrored for Cursor). There is no application to run or build — the "product" is the configuration files themselves.

---

## Repository Structure

**`.claude/rules/`** — Instruction files automatically loaded by Claude Code:
- `testing.md` — Project-specific testing conventions (extends `~/.claude/rules/testing.md`)

**`.claude/agents/`** — Custom subagent definitions (used via the `Agent` tool with `subagent_type`):
- `tdd-workflow.md` — Enforces strict RED → GREEN → REFACTOR with mandatory stops
- `tests-reviewer.md` — Reviews `.spec` files against project testing conventions
- All other agents follow the same frontmatter format (`name`, `description`, then the agent prompt)

**`.claude/skills/`** — User-invocable slash commands (e.g. `/planner`):
- `planner/SKILL.md` — Generates atomic TDD task plans and writes them to `.plans/`

**`.plans/`** — Implementation plans created by `/planner`. Naming: `.plans/<TICKET>:<short-name>.md`

---

## Implementation Approach

**Before starting any implementation**, ask the user:

> "Should we use TDD (Red → Green → Refactor) or implementation-first development?"

If the user chooses **TDD**:
- Use the `tdd-workflow` agent or follow the same cycle manually
- **MANDATORY STOP after RED**: write the failing test, run it, then wait for explicit user approval before proceeding
- **MANDATORY STOP after GREEN**: implement minimal passing code, run tests, then wait for explicit user approval before proceeding
- **MANDATORY STOP after REFACTOR**: clean up, run tests, then wait before moving to the next task
- Valid approval commands: "CONTINUE", "NEXT", "OK", "PROCEED", "yes"
- Never skip a phase or auto-advance past a stop

---

## Adding New Rules, Agents, or Skills

**Rule file** (auto-loaded by Claude Code):
- Place in `.claude/rules/<name>.md`
- No frontmatter required

**Agent file** (invoked via `Agent` tool):
```markdown
---
name: agent-name
description: One-line description used for routing
---
Agent prompt here...
```
Place in `.claude/agents/<name>.md`

**Skill file** (invoked as `/skill-name`):
```markdown
---
name: skill-name
description: ...
---
Skill prompt here...
```
Place in `.claude/skills/<name>/SKILL.md`

---

## TDD & Planning Workflow

The `/planner` skill generates atomic plans following this cycle for every task:

1. 🔴 **RED** — Write failing test → STOP
2. 🟢 **GREEN** — Minimal implementation → STOP
3. 🔵 **REFACTOR** — Clean up → STOP → commit

Each atomic task touches **one file only** (except doc/split tasks). Plans are saved to `.plans/` before execution begins.
