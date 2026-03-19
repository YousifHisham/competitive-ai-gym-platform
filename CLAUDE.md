# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Specify v0.3.2** project — a specification-driven development framework using Claude AI and PowerShell automation. It provides a structured workflow from natural language feature descriptions through specification, planning, task generation, and implementation.

## Speckit Workflow Commands

The project defines custom Claude commands that form a sequential workflow:

1. `/speckit.specify "description"` — Create a feature specification (spec.md)
2. `/speckit.clarify` — Clarify ambiguous requirements in the spec
3. `/speckit.plan "building with [tech stack]"` — Generate a technical implementation plan (plan.md)
4. `/speckit.tasks` — Generate dependency-ordered task list (tasks.md)
5. `/speckit.analyze` — Cross-artifact consistency analysis (read-only)
6. `/speckit.checklist` — Generate quality checklists
7. `/speckit.implement` — Execute implementation from tasks.md
8. `/speckit.taskstoissues` — Export tasks to GitHub issues (requires GitHub MCP server)
9. `/speckit.constitution` — Manage project governance principles

## Architecture

### Directory Structure

- `.claude/commands/` — Claude slash command definitions (the workflow steps above)
- `.specify/scripts/powershell/` — Automation scripts for feature branch management
- `.specify/templates/` — Document templates for specs, plans, tasks, checklists, and agent context
- `.specify/memory/constitution.md` — Project governance principles
- `specs/` — Created per-feature as `specs/###-feature-name/` containing spec.md, plan.md, tasks.md

### PowerShell Scripts

All scripts are in `.specify/scripts/powershell/`:

- **`common.ps1`** — Shared utilities: `Get-RepoRoot`, `Get-CurrentBranch`, `Test-FeatureBranch`, `Get-FeatureDir`, `Get-FeaturePathsEnv`, `Resolve-Template`
- **`create-new-feature.ps1`** — Creates feature branches with auto-numbered naming (`###-feature-name`), generates spec directory and files, outputs JSON
- **`check-prerequisites.ps1`** — Validates feature branch requirements and document availability for each workflow phase, outputs JSON
- **`setup-plan.ps1`** — Initializes planning documents in the feature directory
- **`update-agent-context.ps1`** — Updates AI agent context files (supports Claude, Gemini, Copilot, Cursor) by extracting tech stack from plan.md

### Feature Branch Convention

Feature branches follow the pattern `###-feature-name` (e.g., `001-user-auth`). The next number is auto-detected from existing branches and spec directories.

### Document Flow

Each feature produces artifacts in `specs/###-feature-name/`:
- **spec.md** — User scenarios (P1/P2/P3 priority), requirements, acceptance criteria (Given/When/Then), edge cases
- **plan.md** — Technical context, dependencies, data models, project structure, research phases
- **tasks.md** — Phased task checklist with IDs, priorities, dependencies, and file paths

### Multi-Agent Support

The `update-agent-context.ps1` script generates context files for multiple AI platforms. Templates use a priority resolution stack via `Resolve-Template` in common.ps1.

## No Build/Test/Lint System

This is a framework/template project. Build, test, and lint commands are defined per-feature during implementation. The framework itself consists only of PowerShell scripts, Markdown templates, and Claude command definitions.
