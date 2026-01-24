# Directory Structure

**Analysis Date:** 2026-01-24

## Root Layout

```
get-shit-done/
├── agents/                    # Subagent definitions (11 files)
├── bin/                       # NPM executable entry point
├── commands/                  # Slash commands
│   └── gsd/                   # GSD command namespace (27 files)
├── get-shit-done/             # Core skill content
│   ├── references/            # Shared patterns (9 files)
│   ├── templates/             # Document templates (20+ files)
│   └── workflows/             # Orchestration logic (12 files)
├── hooks/                     # Claude Code hooks
│   ├── dist/                  # Built hooks (gitignored)
│   └── *.js                   # Hook source files
├── scripts/                   # Build scripts
├── .planning/                 # Project state (gitignored locally)
│   └── codebase/              # Codebase analysis docs
├── package.json               # NPM package manifest
├── README.md                  # User documentation
├── CHANGELOG.md               # Version history
├── CONTRIBUTING.md            # Contributor guide
└── MAINTAINERS.md             # Release process
```

## Key Directories

### `/agents/` - Subagent Definitions
Contains markdown files defining specialized Claude subagents.

| File | Purpose |
|------|---------|
| `gsd-executor.md` | Executes plans, manages tasks, commits |
| `gsd-planner.md` | Creates detailed execution plans |
| `gsd-phase-researcher.md` | Researches implementation approaches |
| `gsd-plan-checker.md` | Validates plans before execution |
| `gsd-verifier.md` | Verifies phase goal achievement |
| `gsd-codebase-mapper.md` | Analyzes codebase, writes docs |
| `gsd-debugger.md` | Systematic debugging sessions |
| `gsd-roadmapper.md` | Creates project roadmaps |
| `gsd-project-researcher.md` | Domain research before planning |
| `gsd-research-synthesizer.md` | Synthesizes research outputs |
| `gsd-integration-checker.md` | Verifies cross-phase integration |

### `/commands/gsd/` - Slash Commands
User-invokable commands in Claude Code.

**Core Workflow:**
- `new-project.md` - Initialize project
- `plan-phase.md` - Create phase plans
- `execute-phase.md` - Execute plans

**Project Management:**
- `progress.md` - Check status, route to next action
- `resume-work.md` - Resume from previous session
- `pause-work.md` - Create context handoff

**Phases:**
- `add-phase.md` - Add phase to roadmap end
- `insert-phase.md` - Insert urgent phase between existing
- `remove-phase.md` - Remove future phase

**Research & Analysis:**
- `map-codebase.md` - Analyze existing codebase
- `research-phase.md` - Standalone research
- `discuss-phase.md` - Gather context before planning

**Verification:**
- `verify-work.md` - Validate features via UAT
- `audit-milestone.md` - Audit milestone completion
- `plan-milestone-gaps.md` - Create gap closure plans

**Quick Tasks:**
- `quick.md` - Execute quick task with GSD guarantees

**Utilities:**
- `help.md` - Command reference
- `settings.md` - Configure workflow toggles
- `set-profile.md` - Switch model profile
- `update.md` - Update GSD version
- `join-discord.md` - Community link

### `/get-shit-done/workflows/` - Orchestration Logic
Step-by-step process definitions loaded by commands.

| File | Used By |
|------|---------|
| `new-project.md` | `/gsd:new-project` |
| `plan-phase.md` | `/gsd:plan-phase` |
| `execute-phase.md` | `/gsd:execute-phase` |
| `map-codebase.md` | `/gsd:map-codebase` |
| `progress.md` | `/gsd:progress` |
| `verify-work.md` | `/gsd:verify-work` |
| `audit-milestone.md` | `/gsd:audit-milestone` |
| `complete-milestone.md` | `/gsd:complete-milestone` |

### `/get-shit-done/templates/` - Document Templates
Structure definitions for generated documents.

**Project Level:**
- `project.md` - PROJECT.md template
- `roadmap.md` - ROADMAP.md template
- `requirements.md` - REQUIREMENTS.md template
- `state.md` - STATE.md template

**Phase Level:**
- `phase-prompt.md` - PLAN.md structure
- `summary.md` - SUMMARY.md structure
- `context.md` - CONTEXT.md for decisions
- `verification-report.md` - VERIFICATION.md

**Codebase Analysis:**
- `codebase/stack.md`
- `codebase/architecture.md`
- `codebase/structure.md`
- `codebase/conventions.md`
- `codebase/testing.md`
- `codebase/integrations.md`
- `codebase/concerns.md`

### `/get-shit-done/references/` - Shared Patterns
Cross-cutting conventions loaded via `@` references.

| File | Purpose |
|------|---------|
| `git-integration.md` | Commit conventions, branching |
| `checkpoint-patterns.md` | Human verification gates |
| `questioning.md` | Adaptive questioning methodology |
| `tdd.md` | Test-driven development patterns |
| `ui-brand.md` | GSD branding conventions |
| `state-protocol.md` | STATE.md update patterns |

### `/bin/` - Executable Entry
- `install.js` - NPM package entry point (1,300+ lines)
- Handles: installation, uninstallation, statusline setup
- Supports: Claude Code, OpenCode, global/local installs

### `/hooks/` - Runtime Hooks
- `gsd-statusline.js` - Displays current task + context usage
- `gsd-check-update.js` - Checks for new versions
- `dist/` - Built versions for distribution (gitignored)

## Naming Conventions

**Files:**
- Commands/workflows: `lowercase-with-dashes.md`
- Templates: `lowercase-with-dashes.md` or `UPPERCASE.md`
- Agents: `gsd-{name}.md`
- JavaScript: `lowercase-with-dashes.js`

**Directories:**
- Lowercase with dashes
- Singular for command namespaces (`gsd/`)
- Plural for collections (`commands/`, `agents/`)

## Output Directories

### `.planning/` - Project State
Created per-project, contains all GSD artifacts.

```
.planning/
├── PROJECT.md           # Vision, context, stakeholders
├── ROADMAP.md           # Phase breakdown, status
├── REQUIREMENTS.md      # Checkable requirements
├── STATE.md             # Accumulated decisions, metrics
├── config.json          # Workflow settings
├── research/            # Domain research outputs
├── phases/              # Phase-specific plans and summaries
│   └── XX-name/
│       ├── XX-01-PLAN.md
│       ├── XX-01-SUMMARY.md
│       └── XX-CONTEXT.md
├── codebase/            # Codebase analysis (this directory)
├── quick/               # Quick task plans
└── debug/               # Debug session tracking
```

---

*Structure analysis: 2026-01-24*
