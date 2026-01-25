# Archived Document Template

When archiving documents during a pivot, add this frontmatter and header.

## Frontmatter

```yaml
---
status: archived
authoritative: false
informational_only: true
archived_date: YYYY-MM-DD
archive_version: vN
original_date: YYYY-MM-DD
superseded_by: .planning/[CURRENT_FILE].md
pivot_reason: "[Brief reason for pivot]"
---
```

## Header

Add immediately after frontmatter:

```markdown
# [ARCHIVED] [Original Title]

> **This document is archived and non-authoritative.**
> Current version: See `.planning/[CURRENT_FILE].md`
```

## Field Descriptions

| Field | Required | Description |
|-------|----------|-------------|
| status | Yes | Always `archived` |
| authoritative | Yes | Always `false` |
| informational_only | Yes | Always `true` |
| archived_date | Yes | Date of archival (YYYY-MM-DD) |
| archive_version | Yes | Version directory (v1, v2, etc.) |
| original_date | No | When document was originally created |
| superseded_by | No | Path to current version (if applicable) |
| pivot_reason | No | Brief explanation of why archived |

## Example

```yaml
---
status: archived
authoritative: false
informational_only: true
archived_date: 2026-01-24
archive_version: v1
original_date: 2026-01-10
superseded_by: .planning/PROJECT.md
pivot_reason: "Architecture pivot from monolith to microservices"
---

# [ARCHIVED] Original Project Scope

> **This document is archived and non-authoritative.**
> Current version: See `.planning/PROJECT.md`

[Original content follows...]
```

## Usage

This template is used by `/gsd:pivot` when archiving planning documents.
The frontmatter enables machine-readable status checking.
The header provides clear visual signal for humans.
