---
name: gsd:knowledge
description: Research topics to build structured knowledge repositories with best practices, pitfalls, and debates
argument-hint: "<topic> [--research] [--refresh] [--depth shallow|moderate|deep] [--parallel] [--context <path|query>] [--search <query>] [--list] [--global] [--local] [--format json|markdown|tree]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
  - WebSearch
  - WebFetch
  - AskUserQuestion
  - mcp__context7__*
---

<objective>
Research and organize knowledge on any topic into a hierarchical repository capturing:
- Best practices and idioms
- Common pitfalls and challenges
- Prevailing best-in-practice examples
- Core philosophies and debates
- Alternative approaches and perspectives

Knowledge is stored in a structured hierarchy enabling Claude to filter, traverse, and build context from collected knowledge. Staleness checks (30-day default) ensure knowledge stays current.

**Smart default behavior:**
- Topic exists → Load knowledge (check staleness)
- Topic is new → Launch research workflow
</objective>

<context>
$ARGUMENTS - topic and flags passed to command

Storage locations:
- Project-local (default): `<project-root>/.knowledge/`
- Global/user-level: `~/.claude/.get-shit-done/knowledge/`

Configuration in `.planning/config.json`:
```json
{
  "knowledge": {
    "default_location": "local",
    "staleness_days": 30,
    "default_depth": "moderate"
  }
}
```
</context>

<process>

<step name="parse_arguments">
Extract from $ARGUMENTS:

**Topic** (positional): The subject to research or retrieve
- Example: `"React Hooks"`, `"PostgreSQL Performance"`, `"API Design"`

**Action flags** (mutually exclusive):
- `--research` — Force new research even if topic exists
- `--refresh` — Update existing knowledge (re-research stale sections)
- `--search <query>` — Search across all knowledge
- `--list` — List all topics with metadata
- `--context <path|query>` — Load knowledge for Claude context

**Research control flags:**
- `--depth shallow|moderate|deep` — Research thoroughness (default: moderate)
- `--parallel` — Spawn parallel researcher agents per subtopic

**Storage flags:**
- `--global` — Use `~/.claude/.get-shit-done/knowledge/`
- `--local` — Use `.knowledge/` in project root (default)

**Output flags:**
- `--format json|markdown|tree` — Output format (default: markdown)
- `--quiet` — Minimal output for scripting

**Parse example:**
```bash
TOPIC=""
DEPTH="moderate"
PARALLEL=false
LOCATION="local"
FORMAT="markdown"
ACTION="auto"  # auto, research, refresh, search, list, context

for arg in $ARGUMENTS; do
  case "$arg" in
    --research) ACTION="research" ;;
    --refresh) ACTION="refresh" ;;
    --list) ACTION="list" ;;
    --parallel) PARALLEL=true ;;
    --global) LOCATION="global" ;;
    --local) LOCATION="local" ;;
    --quiet) QUIET=true ;;
    --depth) NEXT_IS_DEPTH=true ;;
    --search) NEXT_IS_SEARCH=true; ACTION="search" ;;
    --context) NEXT_IS_CONTEXT=true; ACTION="context" ;;
    --format) NEXT_IS_FORMAT=true ;;
    shallow|moderate|deep) [ "$NEXT_IS_DEPTH" = true ] && DEPTH="$arg" && NEXT_IS_DEPTH=false ;;
    json|markdown|tree) [ "$NEXT_IS_FORMAT" = true ] && FORMAT="$arg" && NEXT_IS_FORMAT=false ;;
    *)
      if [ "$NEXT_IS_SEARCH" = true ]; then
        SEARCH_QUERY="$arg"; NEXT_IS_SEARCH=false
      elif [ "$NEXT_IS_CONTEXT" = true ]; then
        CONTEXT_PATH="$arg"; NEXT_IS_CONTEXT=false
      elif [ -z "$TOPIC" ]; then
        TOPIC="$arg"
      fi
      ;;
  esac
done
```

If no topic and no action flag:
```
Usage: /gsd:knowledge <topic> [flags]

Actions:
  <topic>                    Research new or load existing topic
  --search <query>           Search across all knowledge
  --list                     List all knowledge topics
  --context <path|query>     Load knowledge into context

Research flags:
  --research                 Force new research
  --refresh                  Update stale knowledge
  --depth shallow|moderate|deep
  --parallel                 Parallelize subtopic research

Storage:
  --local                    .knowledge/ (default)
  --global                   ~/.claude/.get-shit-done/knowledge/

Examples:
  /gsd:knowledge "React Hooks"
  /gsd:knowledge "PostgreSQL" --depth deep --parallel
  /gsd:knowledge --search "caching strategies"
  /gsd:knowledge --context "ReactJS/Hooks/State"
```
Exit.
</step>

<step name="resolve_storage_path">
Determine knowledge base path:

```bash
if [ "$LOCATION" = "global" ]; then
  KNOWLEDGE_BASE="$HOME/.claude/.get-shit-done/knowledge"
else
  KNOWLEDGE_BASE=".knowledge"
fi

# Ensure directory exists
mkdir -p "$KNOWLEDGE_BASE"
```

Read config overrides if available:
```bash
if [ -f ".planning/config.json" ]; then
  CONFIG_LOCATION=$(cat .planning/config.json | grep -o '"default_location"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"')
  CONFIG_STALENESS=$(cat .planning/config.json | grep -o '"staleness_days"[[:space:]]*:[[:space:]]*[0-9]*' | grep -o '[0-9]*$')
  CONFIG_DEPTH=$(cat .planning/config.json | grep -o '"default_depth"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"')

  [ -n "$CONFIG_LOCATION" ] && [ "$LOCATION" = "local" ] && LOCATION="$CONFIG_LOCATION"
  [ -n "$CONFIG_STALENESS" ] && STALENESS_DAYS="$CONFIG_STALENESS" || STALENESS_DAYS=30
  [ -n "$CONFIG_DEPTH" ] && [ "$DEPTH" = "moderate" ] && DEPTH="$CONFIG_DEPTH"
fi
```
</step>

<step name="route_action">
Route based on ACTION:

- `list` → Go to step: list_topics
- `search` → Go to step: search_knowledge
- `context` → Go to step: load_context
- `research` → Go to step: research_topic (force new)
- `refresh` → Go to step: refresh_topic
- `auto` → Go to step: check_existing
</step>

<step name="list_topics">
List all knowledge topics:

```bash
if [ -f "$KNOWLEDGE_BASE/index.json" ]; then
  cat "$KNOWLEDGE_BASE/index.json"
else
  echo "No knowledge base found at $KNOWLEDGE_BASE"
  echo ""
  echo "Start by researching a topic:"
  echo "  /gsd:knowledge \"Your Topic Here\""
fi
```

Format output based on --format flag:

**tree format:**
```
.knowledge/
├── ReactJS/
│   └── Hooks/
│       ├── State/
│       ├── Effects/
│       └── Custom-Hooks/
├── PostgreSQL/
│   └── Performance/
│       ├── Indexing/
│       └── Query-Optimization/
└── API-Design/
    ├── REST/
    └── GraphQL/
```

**markdown format:**
```
# Knowledge Base

| Topic | Depth | Updated | Status |
|-------|-------|---------|--------|
| ReactJS/Hooks | moderate | 5 days ago | ✓ current |
| PostgreSQL/Performance | deep | 45 days ago | ⚠ stale |
| API-Design | shallow | 12 days ago | ✓ current |
```

Exit after display.
</step>

<step name="search_knowledge">
Search across all knowledge:

```bash
# Search file contents
grep -r -l -i "$SEARCH_QUERY" "$KNOWLEDGE_BASE" --include="*.md" 2>/dev/null

# Search topic names
find "$KNOWLEDGE_BASE" -type d -iname "*$SEARCH_QUERY*" 2>/dev/null
```

Display results with context:
```
Search results for: "$SEARCH_QUERY"

## Matching Topics
- PostgreSQL/Performance/Connection-Pooling
- API-Design/Caching-Strategies

## Matching Content
1. ReactJS/Hooks/State/best-practices.md (3 matches)
   Line 45: "...caching state updates..."
   Line 89: "...cache invalidation patterns..."

2. PostgreSQL/Performance/Query-Optimization/examples.md (1 match)
   Line 23: "...query result caching..."

---
Load a result: /gsd:knowledge --context "PostgreSQL/Performance/Connection-Pooling"
```

Exit after display.
</step>

<step name="load_context">
Load knowledge for Claude context building.

**If CONTEXT_PATH looks like a path (contains `/`):**
```bash
CONTEXT_DIR="$KNOWLEDGE_BASE/$CONTEXT_PATH"
if [ -d "$CONTEXT_DIR" ]; then
  # Load all .md files in path
  find "$CONTEXT_DIR" -name "*.md" -type f | while read f; do
    echo "=== $f ==="
    cat "$f"
    echo ""
  done
else
  echo "Path not found: $CONTEXT_PATH"
  echo "Available paths:"
  find "$KNOWLEDGE_BASE" -type d -mindepth 1 | sed "s|$KNOWLEDGE_BASE/||"
fi
```

**If CONTEXT_PATH looks like a query (no `/`):**
Perform semantic search and load matching files:
```bash
# Search and load top matches
grep -r -l -i "$CONTEXT_PATH" "$KNOWLEDGE_BASE" --include="*.md" 2>/dev/null | head -5 | while read f; do
  echo "=== $f ==="
  cat "$f"
  echo ""
done
```

Output loaded content for Claude to use in current context.
Exit after loading.
</step>

<step name="check_existing">
Check if topic already exists:

```bash
# Normalize topic to path format
TOPIC_PATH=$(echo "$TOPIC" | tr ' ' '-' | tr '[:upper:]' '[:lower:]')

# Check for existing directory (case-insensitive)
EXISTING=$(find "$KNOWLEDGE_BASE" -type d -iname "$TOPIC_PATH" 2>/dev/null | head -1)

if [ -n "$EXISTING" ]; then
  echo "existing:$EXISTING"
else
  echo "new"
fi
```

- If existing → Go to step: check_staleness
- If new → Go to step: research_topic
</step>

<step name="check_staleness">
Check if existing knowledge is stale:

```bash
META_FILE="$EXISTING/_meta.json"
if [ -f "$META_FILE" ]; then
  LAST_CHECKED=$(cat "$META_FILE" | grep -o '"last_checked"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"')

  # Calculate days since last check
  LAST_EPOCH=$(date -j -f "%Y-%m-%dT%H:%M:%SZ" "$LAST_CHECKED" "+%s" 2>/dev/null || date -d "$LAST_CHECKED" "+%s")
  NOW_EPOCH=$(date "+%s")
  DAYS_OLD=$(( (NOW_EPOCH - LAST_EPOCH) / 86400 ))

  echo "Days since last check: $DAYS_OLD"
  echo "Staleness threshold: $STALENESS_DAYS"
fi
```

**If stale (DAYS_OLD > STALENESS_DAYS):**

Use AskUserQuestion:
- header: "Stale Knowledge"
- question: "Knowledge for '$TOPIC' is $DAYS_OLD days old (threshold: $STALENESS_DAYS days). What would you like to do?"
- options:
  - "Refresh now" — Re-research and update
  - "Load anyway" — Use existing, update last_checked
  - "Cancel" — Exit

Route based on response:
- Refresh → Go to step: refresh_topic
- Load anyway → Go to step: display_knowledge (update last_checked timestamp)
- Cancel → Exit

**If current:** Go to step: display_knowledge
</step>

<step name="display_knowledge">
Display existing knowledge:

```bash
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo " GSD ► KNOWLEDGE: $TOPIC"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
```

Read and display overview.md first, then list available sections:

```
## Overview
[content from overview.md]

## Available Sections

### State/
- best-practices.md
- pitfalls.md
- examples.md
- debates.md

### Effects/
- best-practices.md
- pitfalls.md

---

► Load section: /gsd:knowledge --context "ReactJS/Hooks/State"
► Refresh: /gsd:knowledge "$TOPIC" --refresh
► Go deeper: /gsd:knowledge "$TOPIC" --depth deep
```

Update last_checked in _meta.json:
```bash
# Update last_checked timestamp
sed -i '' "s/\"last_checked\"[[:space:]]*:[[:space:]]*\"[^\"]*\"/\"last_checked\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"/" "$META_FILE"
```

Exit after display.
</step>

<step name="research_topic">
Launch research workflow for new or forced research.

Display banner:
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► RESEARCHING: $TOPIC
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Depth: $DEPTH
Mode: $([ "$PARALLEL" = true ] && echo "parallel" || echo "sequential")
Storage: $KNOWLEDGE_BASE
```

**Phase 1: Discover Structure**

Use WebSearch and available MCPs to understand topic structure:

```
◆ Phase 1: Discovering structure...
```

Research to identify:
- Main subtopics/categories
- Key concepts to cover
- Authoritative sources

Example output:
```
  → Topic: React Hooks
  → Subtopics discovered:
    - State (useState, useReducer)
    - Effects (useEffect, useLayoutEffect)
    - Context (useContext)
    - Refs (useRef, useImperativeHandle)
    - Custom Hooks
    - Performance (useMemo, useCallback)
```

**Phase 2: Research Subtopics**

For each subtopic, research and compile:
- best-practices.md — Recommended approaches, idioms
- pitfalls.md — Common mistakes, gotchas
- examples.md — Code examples, patterns
- alternatives.md — Different approaches, trade-offs
- debates.md — Conflicting viewpoints, ongoing discussions

**If --parallel flag:**
Spawn parallel Task agents for each subtopic:

```
Task(
  prompt="Research '$SUBTOPIC' within '$TOPIC'. Compile best practices, pitfalls, examples, alternatives. Write findings to $KNOWLEDGE_BASE/$TOPIC_PATH/$SUBTOPIC/",
  subagent_type="gsd-knowledge-researcher",
  description="Research $SUBTOPIC"
)
```

**If sequential (default):**
Research each subtopic depth-first before moving to next.

```
◆ Phase 2: Researching subtopics...
  [1/5] State — best practices, pitfalls, examples...
  [2/5] Effects — best practices, pitfalls, examples...
  [3/5] Context — best practices, pitfalls, examples...
  [4/5] Refs — best practices, pitfalls, examples...
  [5/5] Custom Hooks — best practices, pitfalls, examples...
```

**Depth levels control coverage:**

| Depth | Coverage |
|-------|----------|
| shallow | Overview + best-practices only |
| moderate | All sections, mainstream viewpoints |
| deep | Exhaustive: edge cases, historical context, all alternatives |

Go to step: handle_conflicts
</step>

<step name="handle_conflicts">
Check for conflicting information discovered during research.

```
◆ Phase 3: Checking for conflicts...
```

**If conflicts found:**

For each conflict, use AskUserQuestion:
- header: "Conflict"
- question: "Conflicting information found for: [conflict topic]"
- options:
  - "Document both views" — Capture in debates.md with pros/cons
  - "Prefer authoritative source" — Use official docs over community
  - "Prefer community consensus" — Use widely-adopted practice
  - "Decide later" — Save to _conflicts/ for later resolution

Example conflict:
```
┌─────────────────────────────────────────────────────────────┐
│ CONFLICT: useState vs useReducer preference                 │
├─────────────────────────────────────────────────────────────┤
│ View A (React docs):                                        │
│   "useReducer is preferable for complex state logic"        │
│                                                             │
│ View B (Community):                                         │
│   "Multiple useState calls are cleaner and more readable"   │
├─────────────────────────────────────────────────────────────┤
│ [1] Document both views in debates.md                       │
│ [2] Prefer React docs (authoritative)                       │
│ [3] Prefer community consensus                              │
│ [4] Decide later (save to _conflicts/)                      │
└─────────────────────────────────────────────────────────────┘
```

**If "Decide later":**
Save conflict to `$KNOWLEDGE_BASE/_conflicts/`:
```bash
CONFLICT_FILE="$KNOWLEDGE_BASE/_conflicts/$(date +%Y-%m-%d)-$(echo "$CONFLICT_TOPIC" | tr ' ' '-').md"
cat > "$CONFLICT_FILE" << 'EOF'
---
topic: $TOPIC
subtopic: $SUBTOPIC
detected: $(date -u +%Y-%m-%dT%H:%M:%SZ)
status: pending
---

# Conflict: $CONFLICT_TOPIC

## View A
Source: [source]
Position: [position]

## View B
Source: [source]
Position: [position]

## Notes
[Any additional context]
EOF
```

Go to step: write_knowledge
</step>

<step name="write_knowledge">
Write compiled knowledge to filesystem.

```
◆ Phase 4: Writing knowledge files...
```

**Create directory structure:**
```bash
TOPIC_PATH=$(echo "$TOPIC" | tr ' ' '-')
mkdir -p "$KNOWLEDGE_BASE/$TOPIC_PATH"
```

**Write _meta.json:**
```bash
cat > "$KNOWLEDGE_BASE/$TOPIC_PATH/_meta.json" << EOF
{
  "topic": "$TOPIC_PATH",
  "title": "$TOPIC",
  "created": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "updated": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "last_checked": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "depth": "$DEPTH",
  "staleness_days": $STALENESS_DAYS,
  "sources_used": ["web", "context7"],
  "tags": [],
  "conflicts_pending": false,
  "subtopics": []
}
EOF
```

**Write overview.md:**
```markdown
---
topic: $TOPIC_PATH
section: overview
updated: $(date +%Y-%m-%d)
---

# $TOPIC

## Summary
[High-level overview of the topic]

## Key Concepts
[Core concepts and terminology]

## When to Use
[Guidance on applicability]

## Subtopics
[List of subtopics with brief descriptions]
```

**Write content files for each subtopic:**
Create subdirectories and populate:
- best-practices.md
- pitfalls.md
- examples.md
- alternatives.md (if applicable)
- debates.md (if conflicts documented)

**Update index.json:**
```bash
# Add topic to master index
if [ -f "$KNOWLEDGE_BASE/index.json" ]; then
  # Append to existing index
  # ... JSON manipulation ...
else
  cat > "$KNOWLEDGE_BASE/index.json" << EOF
{
  "version": 1,
  "updated": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "topics": [
    {
      "path": "$TOPIC_PATH",
      "title": "$TOPIC",
      "created": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
      "updated": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
      "depth": "$DEPTH",
      "stale": false
    }
  ]
}
EOF
fi
```

Log files created:
```
  ✓ $TOPIC_PATH/_meta.json
  ✓ $TOPIC_PATH/overview.md
  ✓ $TOPIC_PATH/State/best-practices.md
  ✓ $TOPIC_PATH/State/pitfalls.md
  ...
```

Go to step: display_summary
</step>

<step name="refresh_topic">
Update existing knowledge that is stale.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► REFRESHING: $TOPIC
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

1. Read existing _meta.json to understand current structure
2. Re-research each subtopic looking for:
   - New information since last update
   - Changed best practices
   - Deprecated approaches
   - New debates or resolved conflicts

3. Merge new findings with existing content:
   - Preserve user-resolved conflicts
   - Update examples if APIs changed
   - Add new sections if scope expanded

4. Update timestamps:
```bash
sed -i '' "s/\"updated\"[[:space:]]*:[[:space:]]*\"[^\"]*\"/\"updated\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"/" "$META_FILE"
sed -i '' "s/\"last_checked\"[[:space:]]*:[[:space:]]*\"[^\"]*\"/\"last_checked\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"/" "$META_FILE"
```

5. Check for new conflicts → Go to step: handle_conflicts if found

Go to step: display_summary
</step>

<step name="display_summary">
Display completion summary:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 GSD ► KNOWLEDGE CAPTURED ✓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Topic: $TOPIC
Location: $KNOWLEDGE_BASE/$TOPIC_PATH/
Files: [N] created
Depth: $DEPTH
```

If conflicts saved for later:
```
⚠ [N] conflicts saved to _conflicts/ for resolution
  Review: /gsd:knowledge --list-conflicts
```

Offer next actions:
```
───────────────────────────────────────────────────────────────

## ▶ Next Steps

► View knowledge: /gsd:knowledge "$TOPIC"
► Load into context: /gsd:knowledge --context "$TOPIC_PATH/[subtopic]"
► Go deeper: /gsd:knowledge "$TOPIC" --depth deep
► Search: /gsd:knowledge --search "[query]"

───────────────────────────────────────────────────────────────
```
</step>

</process>

<anti_patterns>
- Don't research without checking for existing knowledge first
- Don't overwrite user-resolved conflicts during refresh
- Don't skip conflict resolution — surface to user
- Don't mix global and local storage in same operation without explicit flags
- Don't research at "deep" depth by default — it's expensive
- Don't ignore staleness warnings — knowledge accuracy matters
</anti_patterns>

<success_criteria>
- [ ] Arguments parsed correctly (topic, flags, storage location)
- [ ] Existing knowledge detected and staleness checked
- [ ] Research depth matches --depth flag
- [ ] Subtopics discovered and researched systematically
- [ ] Conflicts surfaced to user for resolution
- [ ] Knowledge written in correct hierarchical structure
- [ ] _meta.json contains accurate metadata
- [ ] index.json updated with new topic
- [ ] Timestamps reflect actual research/check times
- [ ] Next actions offered to user
</success_criteria>
