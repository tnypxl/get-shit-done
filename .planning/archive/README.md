# Archive - Not Authoritative

This directory contains historical planning artifacts from previous project pivots.

**Status:** Informational only
**Use:** Learning from past decisions, understanding project evolution
**DO NOT:** Reference archived plans as current context

## What Archive Content Is

- Historical record of decisions and rationale
- Lessons learned that may inform new approaches
- Complete snapshots of planning state at pivot time

## What Archive Content Is NOT

- Current planning state
- Authoritative context for agents
- Plans to be executed

## Structure

Each versioned directory (v1/, v2/, etc.) represents a complete planning snapshot at the time of pivot:

```
archive/
├── README.md           # This file
├── v1/                 # First pivot
│   ├── PIVOT.md        # Why we pivoted
│   ├── PROJECT.md      # Old project scope
│   ├── ROADMAP.md      # Old roadmap
│   └── phases/         # Old plans
└── v2/                 # Second pivot (if any)
    └── ...
```

## For Humans

You can browse archive content to:
- Understand why previous approaches were abandoned
- Extract lessons learned
- See evolution of the project

## For Agents

Archive content is explicitly excluded from agent context.
Agents load ONLY `.planning/` files (excluding `.planning/archive/`).
