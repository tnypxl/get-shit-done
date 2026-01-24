# Technology Stack

**Analysis Date:** 2026-01-24

## Primary Language

- **JavaScript (Node.js)** - CommonJS modules
- Version requirement: Node.js ≥16.7.0
- No TypeScript - pure JavaScript with JSDoc type hints

## Runtime

- **Node.js** - Primary execution environment
- Runs within Claude Code or OpenCode CLI
- Interactive installer supports both TTY and non-TTY modes

## Package Manager

- **npm** - Package management
- Uses `package-lock.json` for dependency locking
- Published to npm registry as `get-shit-done-cc`

## Core Dependencies

**Production:**
- None - zero runtime dependencies
- Uses only Node.js built-in modules: `fs`, `path`, `os`, `readline`, `child_process`

**Development:**
- `esbuild ^0.24.0` - Hook bundling/distribution
- No test framework configured

## Build Tools

- `scripts/build-hooks.js` - Copies hooks to `hooks/dist/` for distribution
- `esbuild` - Available but not heavily used (simple copy operation)
- No minification or bundling optimization

## Configuration Files

**Package Manifest:**
- `package.json` - npm package configuration
- Defines `bin` entry point for `npx get-shit-done-cc`
- Scripts: `build`, `prepublishOnly`

**Project Config:**
- `.planning/config.json` - GSD workflow settings
  - `model_profile`: "quality" | "balanced" | "budget"
  - `commit_docs`: boolean (track .planning/ in git)

**Git Config:**
- `.gitignore` - Excludes `node_modules/`, `.claude/`, `hooks/dist/`
- No `.gitattributes` or git hooks

## Installation Methods

**Global (recommended):**
```bash
npx get-shit-done-cc --claude --global
```

**Local (per-project):**
```bash
npx get-shit-done-cc --claude --local
```

**From source:**
```bash
git clone https://github.com/glittercowboy/get-shit-done.git
cd get-shit-done
npm install
```

## Platform Support

- macOS (darwin)
- Windows (win32)
- Linux

Platform-specific handling in `bin/install.js`:
- Path separator normalization
- Home directory detection (`os.homedir()`)
- Hook command format differences

## File Structure Conventions

**Executable Scripts:**
- `bin/install.js` - NPM package entry point
- `hooks/gsd-statusline.js` - Statusline hook
- `hooks/gsd-check-update.js` - Update checker hook

**Markdown Assets:**
- `commands/gsd/*.md` - Slash commands (27 files)
- `agents/*.md` - Subagent definitions (11 files)
- `get-shit-done/workflows/*.md` - Workflow orchestration
- `get-shit-done/templates/*.md` - Document templates
- `get-shit-done/references/*.md` - Shared patterns

---

*Stack analysis: 2026-01-24*
