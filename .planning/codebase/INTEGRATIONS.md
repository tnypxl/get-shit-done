# External Integrations

**Analysis Date:** 2026-01-24

## Overview

GSD is a meta-prompting framework with minimal external dependencies. It operates entirely within Claude Code/OpenCode runtime and uses the local filesystem for state management.

## Runtime Integrations

### Claude Code
- **Purpose:** Primary execution runtime
- **Integration:** Custom commands loaded from `~/.claude/commands/gsd/`
- **Hooks:** Statusline display, update checking via hooks system
- **API:** Uses Claude Code's Task tool for subagent spawning

### OpenCode
- **Purpose:** Alternative execution runtime
- **Integration:** Commands loaded from `~/.opencode/command/gsd-*.md`
- **Differences:** Flat command structure, different frontmatter format
- **Conversion:** `convertClaudeToOpencodeFrontmatter()` in installer

## Distribution

### npm Registry
- **Package:** `get-shit-done-cc`
- **Purpose:** Distribution and installation
- **Entry:** `npx get-shit-done-cc` triggers installer
- **Versions:** Semantic versioning with prerelease support

### GitHub
- **Repository:** `glittercowboy/get-shit-done`
- **Purpose:** Source control, releases, issue tracking
- **CI/CD:** GitHub Actions for release automation
- **Releases:** Auto-created from CHANGELOG.md on version tags

## Local Integrations

### Git
- **Purpose:** Version control, atomic commits per task
- **Integration:** Direct `git` CLI calls from workflows
- **Patterns:**
  - Conventional commits: `feat(XX-YY): task-name`
  - Planning docs optionally committed (`commit_docs` config)
  - Auto-detection of gitignored `.planning/` directory

### Filesystem
- **State Storage:** `.planning/` directory tree
  - `PROJECT.md`, `ROADMAP.md`, `STATE.md`
  - `phases/XX-name/` per phase
  - `research/` for domain research
  - `codebase/` for codebase analysis
- **Config:** `.planning/config.json`
- **Cache:** `~/.claude/cache/` (update check cache)

## No External APIs

This project does not integrate with:
- Databases (no SQLite, PostgreSQL, etc.)
- Authentication providers (no OAuth, JWT)
- Cloud services (no AWS, Vercel, etc.)
- Third-party APIs (no Stripe, SendGrid, etc.)

## Webhook/Event Patterns

**Claude Code Hooks:**
- `PreToolUse` - Not used
- `PostToolUse` - Not used
- `Stop` - Statusline display hook
- `Notification` - Update check hook

**Hook Registration:**
```json
{
  "hooks": {
    "Stop": [{
      "type": "command",
      "command": "node ~/.claude/hooks/dist/gsd-statusline.js"
    }]
  }
}
```

## Environment Variables

**Used by installer:**
- `HOME` / `USERPROFILE` - Home directory detection
- None required at runtime

**Not used:**
- No API keys
- No secrets management
- No environment-specific configuration

## Development Integrations

### Discord
- **Purpose:** Community support and updates
- **Integration:** Links in documentation
- **Command:** `/gsd:join-discord` provides invite link

---

*Integrations analysis: 2026-01-24*
