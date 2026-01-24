# Architecture

**Analysis Date:** 2026-01-24

## Pattern Overview

**Overall:** Meta-Prompting Framework with Orchestrator-Subagent Architecture

**Key Characteristics:**
- Command-based interface (slash commands) that loads context via `@` references
- Orchestrator-subagent pattern for parallel execution and fresh context per task
- State-driven workflow with persistent context accumulation across sessions
- Document-as-prompt design (templates ARE the execution context, not inputs to prompts)
- Git-integrated with atomic commits per task for reproducible state snapshots

## Layers

### Commands Layer
- **Purpose:** User-facing slash commands that orchestrate workflows
- **Location:** `commands/gsd/`
- **Contains:** Markdown command definitions with frontmatter (name, description, allowed tools, arguments)
- **Depends on:** Workflows, agents, templates, references
- **Used by:** Claude Code/OpenCode runtime (loaded as custom commands)

### Workflows Layer
- **Purpose:** Detailed process steps for executing commands (orchestrator logic)
- **Location:** `get-shit-done/workflows/`
- **Contains:** Step-by-step orchestration instructions loaded via `@execution_context`
- **Depends on:** Agents, templates, references
- **Used by:** Commands layer (via `@` references in command markdown)

### Agents Layer
- **Purpose:** Specialized subagent prompts for focused tasks (planning, execution, verification)
- **Location:** `agents/`
- **Contains:** Markdown agent definitions with role, philosophy, process, tools
- **Depends on:** Templates, references, workflows
- **Used by:** Workflows via Task tool with `subagent_type` parameter

### Templates Layer
- **Purpose:** Document structure definitions for artifacts created during workflows
- **Location:** `get-shit-done/templates/`
- **Contains:** Markdown templates for PLAN.md, SUMMARY.md, PROJECT.md, STATE.md, etc.
- **Depends on:** References (for formatting guidelines)
- **Used by:** Agents and workflows when writing output documents

### References Layer
- **Purpose:** Cross-cutting patterns and conventions shared across workflows
- **Location:** `get-shit-done/references/`
- **Contains:** Git integration rules, checkpoint patterns, TDD methodology, UI branding
- **Depends on:** Nothing (foundation layer)
- **Used by:** Commands, workflows, agents via `@` references

### Installation Layer
- **Purpose:** NPM package installation and hook deployment
- **Location:** `bin/install.js`, `hooks/`, `scripts/build-hooks.js`
- **Contains:** Interactive installer, statusline hook, update checker
- **Depends on:** Package manifest
- **Used by:** `npx get-shit-done-cc` (installation entry point)

## Data Flow

### Project Initialization Flow
1. User runs `/gsd:new-project`
2. Command loads `@~/.claude/get-shit-done/references/questioning.md`
3. Orchestrator detects brownfield code → offers `/gsd:map-codebase`
4. Creates `.planning/PROJECT.md`, `.planning/ROADMAP.md`, `.planning/STATE.md`
5. Git commits initialization with structured message

### Phase Planning Flow
1. User runs `/gsd:plan-phase 1`
2. Orchestrator spawns `gsd-phase-researcher` agent (if needed)
3. Researcher writes `.planning/research/phase-XX-RESEARCH.md`
4. Orchestrator spawns `gsd-planner` agent with research context
5. Planner creates `.planning/phases/XX-name/XX-01-PLAN.md` (multiple plans per phase)
6. Orchestrator spawns `gsd-plan-checker` agent for verification loop
7. Checker validates plans, returns issues or approval
8. Iterate planner → checker until plans pass or max iterations
9. Git commits plans (if `commit_docs: true`)

### Phase Execution Flow
1. User runs `/gsd:execute-phase 1`
2. Orchestrator reads plans from `.planning/phases/01-name/`
3. Groups plans into waves based on `wave` frontmatter (dependency-ordered)
4. For each wave, spawns parallel `gsd-executor` agents
5. Each executor reads PLAN.md, executes tasks, commits per task
6. Executor writes `XX-01-SUMMARY.md` on completion
7. Updates `.planning/STATE.md` with progress metrics
8. Git commits summary + state update

### Verification Flow
1. User runs `/gsd:verify-work`
2. Orchestrator spawns `gsd-verifier` agent
3. Verifier reads completed plans' `must_haves` (goal-backward criteria)
4. Runs verification commands, checks artifacts, validates key links
5. Writes `VERIFICATION.md` with pass/fail per plan
6. On failures, user runs `/gsd:plan-phase --gaps` → creates gap closure plans

## Key Abstractions

### Command Abstraction
- **Purpose:** User interface to GSD system
- **Examples:** `commands/gsd/execute-phase.md`, `commands/gsd/plan-phase.md`
- **Pattern:** Frontmatter (metadata) + `<objective>` + `<execution_context>` + `<process>`

### Agent Abstraction
- **Purpose:** Specialized Claude instances with fresh context per task
- **Examples:** `agents/gsd-executor.md`, `agents/gsd-planner.md`, `agents/gsd-verifier.md`
- **Pattern:** `<role>` + `<philosophy>` + `<process>` + tool definitions

### Plan Abstraction
- **Purpose:** Executable prompts (not documents that become prompts)
- **Examples:** `.planning/phases/01-foundation/01-01-PLAN.md`
- **Pattern:** Frontmatter (wave, depends_on, must_haves) + `<objective>` + `<context>` + `<tasks>` + `<verification>`

### Task Abstraction
- **Purpose:** Atomic unit of work with verification
- **Examples:** `<task type="auto">`, `<task type="checkpoint:human-verify">`
- **Pattern:** `<name>` + `<files>` + `<action>` + `<verify>` + `<done>`

### @ Reference Abstraction
- **Purpose:** Context loading mechanism (Claude Code feature)
- **Examples:** `@.planning/STATE.md`, `@~/.claude/get-shit-done/workflows/execute-phase.md`
- **Pattern:** File path prefixed with `@` loads content into prompt context

## Entry Points

### NPM Package Entry
- **Location:** `bin/install.js`
- **Triggers:** `npx get-shit-done-cc` or `npx get-shit-done-cc --claude --global`
- **Responsibilities:** Interactive or non-interactive installation, copy files to `~/.claude/` or `./.claude/`

### Command Entry Points
- **Location:** `commands/gsd/*.md`
- **Triggers:** User types `/gsd:command-name` in Claude Code
- **Responsibilities:** Load execution context, validate environment, orchestrate workflows

### Hook Entry Points
- **Location:** `hooks/gsd-statusline.js`, `hooks/gsd-check-update.js`
- **Triggers:** Claude Code runtime (statusline on every render, update check periodically)
- **Responsibilities:** Display context usage + current task, check for new versions

## State Management

- `.planning/STATE.md` accumulates decisions, blockers, metrics across all phases
- `.planning/ROADMAP.md` tracks phase status and plan completion
- Each plan SUMMARY feeds into STATE.md for session continuity
- `config.json` controls behavior (model_profile, commit_docs)

## Error Handling

**Strategy:** Fail fast with actionable messages + optional recovery paths

**Patterns:**
- Environment validation at start of each workflow (check `.planning/` exists, git repo initialized)
- Model profile resolution with fallback to "balanced" if config missing
- Graceful degradation when optional features unavailable (e.g., WebFetch for research)
- Checkpoint gates (`gate="blocking"`) halt execution until user input received
- Verification loops with max iterations (planner ↔ checker, executor ↔ verifier)

## Cross-Cutting Concerns

**Context Management:**
- Model profiles control which Claude model executes each agent (`quality`/`balanced`/`budget`)
- Wave-based parallelization keeps executor context fresh (multiple agents in background)
- Orchestrators stay lean by delegating to subagents (15% orchestrator context, 100% per subagent)
- `/clear` recommendations in continuation format to reset context window

**Git Integration:**
- Atomic commits per task (not per file)
- Conventional commit format (`feat(XX-YY): task-name`)
- Planning docs committed only if `commit_docs: true` in `config.json`
- Auto-detection of gitignored `.planning/` overrides config

---

*Architecture analysis: 2026-01-24*
