# Testing

**Analysis Date:** 2026-01-24

## Framework

**Status:** No test framework configured

This project does not have automated tests. Key observations:

- No test runner (no Jest, Vitest, Mocha, etc.)
- No test files (`*.test.js`, `*.spec.js`)
- No test configuration files
- No test scripts in `package.json`
- No coverage reporting

## Why No Tests?

This is a **meta-prompting framework**, not a traditional application:

1. **Content-driven:** Most of the codebase is markdown prompt templates
2. **Runtime-dependent:** Functionality depends on Claude Code/OpenCode runtime
3. **Installation-focused:** The main JavaScript code is the installer
4. **Stateless prompts:** Commands/agents are declarative, not imperative

## What Could Be Tested

### Installer Logic (`bin/install.js`)
- Path resolution across platforms
- File copying and transformation
- Settings file manipulation
- Frontmatter conversion (Claude → OpenCode)
- Cleanup of orphaned files

### Hook Logic (`hooks/*.js`)
- JSON parsing and error handling
- Output formatting
- Cache management (update checker)

### Build Scripts (`scripts/build-hooks.js`)
- File copying to dist/

## Current Quality Assurance

Instead of automated tests, this project relies on:

1. **Manual testing** during development
2. **User feedback** via GitHub issues
3. **Incremental releases** with semantic versioning
4. **Changelogs** documenting behavior changes
5. **Community testing** via Discord

## Test-Related Patterns in Codebase

The project does define **TDD patterns** for user projects:

- `get-shit-done/references/tdd.md` - Test-driven development reference
- Verification steps in plan templates
- `<verify>` blocks in task definitions

These are patterns for **projects using GSD**, not for testing GSD itself.

## Recommendations for Adding Tests

If tests were to be added:

1. **Framework:** Vitest (already have esbuild dependency)
2. **Focus areas:**
   - `bin/install.js` - Core installer logic
   - `hooks/*.js` - Hook behavior
   - Path resolution edge cases
3. **Mocking:** Would need to mock `fs`, `os`, `readline`
4. **Fixtures:** Sample settings files, hook inputs

## Dependencies Related to Testing

```json
{
  "devDependencies": {
    "esbuild": "^0.24.0"
  }
}
```

Only esbuild is installed - no test framework or assertion library.

---

*Testing analysis: 2026-01-24*
