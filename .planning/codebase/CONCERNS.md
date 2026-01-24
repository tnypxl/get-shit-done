# Concerns

**Analysis Date:** 2026-01-24

## Technical Debt

### No Test Coverage
- **Severity:** Medium
- **Location:** Entire codebase
- **Issue:** Zero automated tests for 1,300+ line installer
- **Impact:** Regressions possible with changes
- **Suggested fix:** Add Vitest tests for installer core functions

### Silent Error Handling
- **Severity:** Medium
- **Location:** `hooks/gsd-statusline.js:58,71`, `hooks/gsd-check-update.js:41,46`
- **Issue:** Empty catch blocks swallow errors silently
- **Impact:** Failures invisible to users, debugging difficult
- **Suggested fix:** Log errors to stderr or debug file

### Large Monolithic Installer
- **Severity:** Low
- **Location:** `bin/install.js` (1,300+ lines, 24+ functions)
- **Issue:** Single file handles all installation logic
- **Impact:** Difficult to maintain and test
- **Suggested fix:** Split into modules (path-utils, settings, copy-logic)

## Known Issues

### Hardcoded Orphan Lists
- **Severity:** Low
- **Location:** `bin/install.js:477-480,495-500`
- **Issue:** Deprecated files hardcoded in arrays
- **Impact:** New deprecated files won't auto-cleanup
- **Suggested fix:** Use manifest file or pattern matching

### Partial Install Recovery
- **Severity:** Medium
- **Location:** `bin/install.js:443-445`
- **Issue:** Clean install deletes directories before copying
- **Impact:** Failure mid-install leaves broken state
- **Suggested fix:** Copy to temp then atomic rename

## Security Considerations

### Unvalidated User Paths
- **Severity:** Low
- **Location:** `bin/install.js` - `--config-dir` handling
- **Issue:** User-provided paths used without sanitization
- **Impact:** Potential path traversal (low risk in CLI context)
- **Suggested fix:** Validate paths are within expected directories

### Synchronous File Operations
- **Severity:** Low
- **Location:** Throughout `bin/install.js`
- **Issue:** Uses sync fs methods without validation
- **Impact:** No async error boundaries
- **Note:** Acceptable for CLI installer

## Performance Considerations

### Update Check Blocking
- **Severity:** Low
- **Location:** `hooks/gsd-check-update.js`
- **Issue:** Update check runs in background but has 10s timeout
- **Impact:** Could delay hook completion
- **Note:** Mitigated by spawn with detached option

### No Caching Strategy
- **Severity:** Low
- **Issue:** File system operations not optimized
- **Impact:** Slightly slower than necessary
- **Note:** Acceptable given infrequent execution

## Fragile Areas

### Platform-Specific Paths
- **Files:** `bin/install.js:185-198,40-70`
- **Issue:** Complex path handling for Windows/macOS/Linux
- **Risk:** Edge cases on unusual system configurations
- **Testing:** Manual testing on each platform required

### OpenCode Compatibility
- **Files:** `bin/install.js:261-382`
- **Issue:** Frontmatter conversion between Claude/OpenCode formats
- **Risk:** Changes to either runtime could break conversion
- **Testing:** Must test on both runtimes after changes

### Settings File Format
- **Files:** `bin/install.js:205-256`
- **Issue:** Directly manipulates JSON settings files
- **Risk:** Format changes in Claude Code could break installation
- **Testing:** Verify against actual Claude Code settings

## Breaking Changes History

Notable breaking changes that could affect users:

| Version | Change |
|---------|--------|
| v1.9.2 | Removed SQLite-based codebase intelligence |
| v1.9.0 | Changed command structure, removed deprecated commands |
| v1.6.0 | Major workflow restructuring |
| v1.5.28 | Hook installation changes |

## Missing Features

### No Rollback Mechanism
- Can't undo installation to previous version
- Workaround: Manual git checkout and reinstall

### No Offline Support
- Update checker requires network
- Workaround: Fails silently, doesn't block usage

### No Verbose/Debug Mode
- Limited visibility into installation process
- Workaround: Read source code

## Monitoring Points

If monitoring were added, focus on:

1. **Installation success rate** - Track failures by platform
2. **Hook execution time** - Monitor statusline performance
3. **Update check latency** - Network dependency
4. **Common error paths** - Which failures occur most

---

*Concerns analysis: 2026-01-24*
