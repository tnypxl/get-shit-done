# Coding Conventions

**Analysis Date:** 2026-01-24

## Naming Patterns

### Files
- Markdown files: UPPERCASE.md for templates/documents (`PROJECT.md`, `ROADMAP.md`, `STATE.md`)
- Markdown files: lowercase-with-dashes.md for commands and workflows (`new-milestone.md`, `execute-phase.md`)
- JavaScript files: lowercase-with-dashes.js for executables (`gsd-statusline.js`, `gsd-check-update.js`)
- Agent files: gsd-{name}.md pattern (`gsd-executor.md`, `gsd-planner.md`, `gsd-verifier.md`)
- Command files: gsd/{command-name}.md in commands directory

### Functions
- camelCase for all functions
- Descriptive verb-noun pattern: `getDirName()`, `expandTilde()`, `parseConfigDirArg()`
- Function names communicate purpose: `buildHookCommand()`, `readSettings()`, `writeSettings()`

### Variables
- camelCase for regular variables: `hasGlobal`, `isOpencode`, `settingsPath`
- UPPER_SNAKE_CASE for shell/environment variables in bash blocks: `MODEL_PROFILE`, `COMMIT_PLANNING_DOCS`, `PLAN_START_TIME`
- Constants use const keyword with descriptive names: `HOOKS_DIR`, `DIST_DIR`, `HOOKS_TO_COPY`
- Boolean variables prefixed with `has` or `is`: `hasGlobal`, `hasLocal`, `isOpencode`, `hasHelp`

### Types
- JSDoc comments document parameter types: `@param {string} runtime`, `@param {string|null} explicitDir`
- No TypeScript - pure JavaScript with CommonJS modules

## Code Style

### Formatting
- No automatic formatter detected (no .prettierrc or .eslintrc)
- 2-space indentation consistently used across all JavaScript files
- Single quotes for strings in JavaScript
- Backticks for template literals and multi-line strings

### Linting
- No ESLint configuration detected
- No automated linting in npm scripts

### Language Features
- CommonJS modules: `require()` and `module.exports` (not ES6 imports)
- Uses Node.js built-in modules: `fs`, `path`, `os`, `readline`
- Modern JavaScript: arrow functions, template literals, ternary operators
- Strict equality: `===` and `!==` used throughout (never `==` or `!=`)

## Import Organization

**Order:**
1. Node.js built-in modules (`fs`, `path`, `os`, `readline`)
2. npm package imports (`require('../package.json')`)
3. Local constants and configuration

**Pattern in `bin/install.js`:**
```javascript
const fs = require('fs');
const path = require('path');
const os = require('os');
const readline = require('readline');

// Colors
const cyan = '\x1b[36m';
const green = '\x1b[32m';

// Get version from package.json
const pkg = require('../package.json');
```

**Path Aliases:**
- No path aliases - uses relative paths (`../package.json`, `path.join()`)

## Error Handling

### Patterns
- Try-catch blocks used sparingly - mostly for JSON parsing
- Silent failure pattern: Empty catch blocks that allow execution to continue
- File existence checks before operations: `fs.existsSync()` before reading
- Default values using OR operator: `data.model?.display_name || 'Claude'`
- Optional chaining: `data.model?.display_name`
- No explicit throw statements - errors handled locally

**Example from `hooks/gsd-statusline.js`:**
```javascript
try {
  const data = JSON.parse(input);
  // ... process data
} catch (e) {
  // Silent fail - don't break statusline on parse errors
}
```

## Logging

**Framework:** Native `console` methods (no logging library)

**Patterns:**
- `console.log()` for user-facing output with color codes
- `console.error()` for error messages
- `console.warn()` for warnings in build scripts
- Color-coded output using ANSI escape codes: `\x1b[36m` (cyan), `\x1b[32m` (green), `\x1b[33m` (yellow)
- Status symbols: `✓` for success, `✗` for failure, `⚠` for warnings
- Structured output with consistent formatting

## Comments

### When to Comment
- File-level comments for executables: `#!/usr/bin/env node` followed by description
- JSDoc-style comments for complex functions with parameters
- Inline comments for non-obvious logic or important context
- Section dividers in long files: `// Colors`, `// Parse args`, `// Runtime selection`

### JSDoc/TSDoc
- Used for functions with parameters
- Documents param types and purpose
- Multi-line format with `@param` tags

**Example from `bin/install.js`:**
```javascript
/**
 * Get the global config directory for a runtime
 * @param {string} runtime - 'claude' or 'opencode'
 * @param {string|null} explicitDir - Explicit directory from --config-dir flag
 */
function getGlobalDir(runtime, explicitDir = null) {
```

## Markdown Structure

### Frontmatter
- YAML frontmatter at start of command/agent files
- Fields: `name:`, `description:`, `allowed-tools:`, `argument-hint:`, `tools:`, `color:`
- Colon-prefixed format for commands: `name: gsd:new-milestone`
- Agent names without prefix: `name: gsd-executor`

### XML-Style Tags
- Custom XML tags for prompt structure: `<objective>`, `<context>`, `<process>`, `<step>`, `<role>`
- Used extensively in command and agent files for Claude prompt engineering
- Self-closing and paired tags: `<step name="load_plan">...</step>`

### Code Blocks
- Triple backtick fences with language hints: ` ```bash`, ` ```javascript`, ` ```markdown`
- Bash blocks for shell commands in workflows
- Markdown blocks for template output examples

### File References
- `@` symbol for file references in context sections: `@.planning/PROJECT.md`
- Tilde expansion for global paths: `@~/.claude/get-shit-done/references/ui-brand.md`

## Function Design

### Size
- Functions range from 5-150 lines
- Long functions handle complex workflows (installer logic)
- Small utility functions for single-purpose operations

### Parameters
- Default parameters used: `function getGlobalDir(runtime, explicitDir = null)`
- Named parameters with descriptive names
- Optional parameters come last
- Runtime parameter for multi-platform support

### Return Values
- Explicit returns for pure functions
- No return (undefined) for side-effect functions (installers, loggers)
- Early returns for validation: `if (!fs.existsSync(dir)) return;`

## Module Design

### Exports
- No exports from scripts - designed as executables
- Standalone scripts with immediate execution
- Self-contained functions within files

### Barrel Files
- Not used - flat file structure

## Shell Script Patterns

### Bash Commands
- Used within markdown code blocks for workflow automation
- Pattern: Variable capture with command substitution: `VAR=$(command)`
- Default values: `VAR=$(command || echo "default")`
- JSON extraction with grep: `grep -o '"field"[[:space:]]*:[[:space:]]*[^,}]*'`
- Git operations wrapped in commit checks: `git check-ignore -q .planning`

**Example from `commands/gsd/quick.md`:**
```bash
MODEL_PROFILE=$(cat .planning/config.json 2>/dev/null | grep -o '"model_profile"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "balanced")
```

---

*Convention analysis: 2026-01-24*
