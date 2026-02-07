<purpose>
Execute all plans in a phase using wave-based parallel execution. Orchestrator stays lean — delegates plan execution to subagents.
</purpose>

<core_principle>
Orchestrator coordinates, not executes. Each subagent loads the full execute-plan context. Orchestrator: discover plans → analyze deps → group waves → spawn agents → handle checkpoints → collect results.
</core_principle>

<required_reading>
Read STATE.md and config.json before any operation.
</required_reading>

<process>

<step name="resolve_model_profile" priority="first">

```bash
EXECUTOR_MODEL=$(node ~/.claude/get-shit-done/bin/gsd-tools.js resolve-model gsd-executor --raw)
VERIFIER_MODEL=$(node ~/.claude/get-shit-done/bin/gsd-tools.js resolve-model gsd-verifier --raw)
```

</step>

<step name="load_project_state">

```bash
cat .planning/STATE.md 2>/dev/null
```

**If exists:** Parse current position, accumulated decisions, blockers.
**If missing but .planning/ exists:** Offer reconstruct from artifacts or continue without state.
**If .planning/ missing:** Error — project not initialized.

**Load configs:**

```bash
GSD_CONFIG=$(node ~/.claude/get-shit-done/bin/gsd-tools.js state load --raw)
COMMIT_PLANNING_DOCS=$(echo "$GSD_CONFIG" | grep '^commit_docs=' | cut -d= -f2)
PARALLELIZATION=$(echo "$GSD_CONFIG" | grep '^parallelization=' | cut -d= -f2)
BRANCHING_STRATEGY=$(echo "$GSD_CONFIG" | grep '^branching_strategy=' | cut -d= -f2)
PHASE_BRANCH_TEMPLATE=$(echo "$GSD_CONFIG" | grep '^phase_branch_template=' | cut -d= -f2)
MILESTONE_BRANCH_TEMPLATE=$(echo "$GSD_CONFIG" | grep '^milestone_branch_template=' | cut -d= -f2)
```

When `PARALLELIZATION=false`, plans within a wave execute sequentially.

**Read check-in config:**

```bash
# Read check-in granularity (default: phase)
CHECKIN_GRANULARITY=$(cat .planning/config.json 2>/dev/null | grep -o '"checkin_granularity"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "phase")

# Read pause on failure (default: true)
PAUSE_ON_FAILURE=$(cat .planning/config.json 2>/dev/null | grep -o '"pause_on_failure"[[:space:]]*:[[:space:]]*[^,}]*' | grep -o 'true\|false' || echo "true")

# Read execution mode for YOLO bypass
CURRENT_MODE=$(cat .planning/config.json 2>/dev/null | grep -o '"mode"[[:space:]]*:[[:space:]]*"[^"]*"' | grep -o '"[^"]*"$' | tr -d '"' || echo "interactive")
```

**Save global value before per-phase override:**

```bash
GLOBAL_GRANULARITY="$CHECKIN_GRANULARITY"  # Save before per-phase override
```

**Read per-phase granularity override:**

```bash
# Read per-phase granularity from ROADMAP.md (if present, overrides global)
PHASE_GRANULARITY=$(sed -n "/### Phase.*${PHASE_NUMBER}.*:/,/### Phase/p" .planning/ROADMAP.md 2>/dev/null | grep -o '\*\*Granularity\*\*:[[:space:]]*[a-z]*' | grep -o '[a-z]*$' || echo "")

if [ -n "$PHASE_GRANULARITY" ]; then
  case "$PHASE_GRANULARITY" in
    wave|plan|none) CHECKIN_GRANULARITY="$PHASE_GRANULARITY" ;;
    *) echo "WARNING: Invalid per-phase granularity '$PHASE_GRANULARITY' in ROADMAP.md — using global: $CHECKIN_GRANULARITY" ;;
  esac
fi
```

**YOLO bypass (runs LAST — wins always):**

```bash
# YOLO bypass: no check-ins, no failure pausing
if [ "$CURRENT_MODE" = "yolo" ]; then
  CHECKIN_GRANULARITY="phase"
  PAUSE_ON_FAILURE="false"
fi
```

Store `CHECKIN_GRANULARITY`, `PAUSE_ON_FAILURE`, `CURRENT_MODE`, and `GLOBAL_GRANULARITY` for use in execute_waves.
</step>

<step name="handle_branching">
Create or switch to branch based on `BRANCHING_STRATEGY`.

**"none":** Skip, continue on current branch.

**"phase":**
```bash
PHASE_NAME=$(basename "$PHASE_DIR" | sed 's/^[0-9]*-//')
PHASE_SLUG=$(node ~/.claude/get-shit-done/bin/gsd-tools.js generate-slug "$PHASE_NAME" --raw)
BRANCH_NAME=$(echo "$PHASE_BRANCH_TEMPLATE" | sed "s/{phase}/$PADDED_PHASE/g" | sed "s/{slug}/$PHASE_SLUG/g")
git checkout -b "$BRANCH_NAME" 2>/dev/null || git checkout "$BRANCH_NAME"
```

**"milestone":**
```bash
MILESTONE_VERSION=$(grep -oE 'v[0-9]+\.[0-9]+' .planning/ROADMAP.md | head -1 || echo "v1.0")
MILESTONE_NAME=$(grep -A1 "## .*$MILESTONE_VERSION" .planning/ROADMAP.md | tail -1 | sed 's/.*- //' | cut -d'(' -f1 | tr -d ' ' || echo "milestone")
MILESTONE_SLUG=$(node ~/.claude/get-shit-done/bin/gsd-tools.js generate-slug "$MILESTONE_NAME" --raw)
BRANCH_NAME=$(echo "$MILESTONE_BRANCH_TEMPLATE" | sed "s/{milestone}/$MILESTONE_VERSION/g" | sed "s/{slug}/$MILESTONE_SLUG/g")
git checkout -b "$BRANCH_NAME" 2>/dev/null || git checkout "$BRANCH_NAME"
```

All subsequent commits go to this branch. User handles merging.
</step>

<step name="validate_phase">

```bash
PHASE_INFO=$(node ~/.claude/get-shit-done/bin/gsd-tools.js find-phase "${PHASE_ARG}")
PHASE_DIR=$(echo "$PHASE_INFO" | grep -o '"directory":"[^"]*"' | cut -d'"' -f4)
if [ -z "$PHASE_DIR" ]; then
  echo "ERROR: No phase directory matching '${PHASE_ARG}'"
  exit 1
fi

PLAN_COUNT=$(ls -1 "$PHASE_DIR"/*-PLAN.md 2>/dev/null | wc -l | tr -d ' ')
if [ "$PLAN_COUNT" -eq 0 ]; then
  echo "ERROR: No plans found in $PHASE_DIR"
  exit 1
fi
```

Report: "Found {N} plans in {phase_dir}"

If per-phase override is active (CHECKIN_GRANULARITY differs from GLOBAL_GRANULARITY and CURRENT_MODE is not "yolo"):
  Report: "Phase {N} granularity: {CHECKIN_GRANULARITY} (overrides global: {GLOBAL_GRANULARITY})"

Only announce when an override is active. No announcement when using global config.
</step>

<step name="discover_plans">

```bash
ls -1 "$PHASE_DIR"/*-PLAN.md 2>/dev/null | sort
ls -1 "$PHASE_DIR"/*-SUMMARY.md 2>/dev/null | sort
```

For each plan, read frontmatter: `wave`, `autonomous`, `gap_closure`.

Build inventory: path, plan ID, wave, autonomous flag, gap_closure flag, completion (SUMMARY exists = complete).

**Filtering:** Skip completed plans. If `--gaps-only`: also skip non-gap_closure plans. If all filtered: "No matching incomplete plans" → exit.
</step>

<step name="group_by_wave">

```bash
for plan in $PHASE_DIR/*-PLAN.md; do
  wave=$(grep "^wave:" "$plan" | cut -d: -f2 | tr -d ' ')
  autonomous=$(grep "^autonomous:" "$plan" | cut -d: -f2 | tr -d ' ')
  echo "$plan:$wave:$autonomous"
done
```

Group by wave number. **No dependency analysis needed** — waves pre-computed during `/gsd:plan-phase`.

Report:
```
## Execution Plan

**Phase {X}: {Name}** — {total_plans} plans across {wave_count} waves

| Wave | Plans | What it builds |
|------|-------|----------------|
| 1 | 01-01, 01-02 | {from plan objectives, 3-8 words} |
| 2 | 01-03 | ... |
```
</step>

<step name="checkin_interaction">
Reusable check-in interaction loop. Called by execute_waves at wave, plan, failure, or phase boundaries.

**Inputs:** check-in type ("wave", "plan", "failure", or "phase"), status summary text, completed plan IDs, check-in context (wave number or plan ID).

**Adaptive density:**
Before presenting, evaluate density:
- Read SUMMARY.md for each just-completed plan
- EXPANDED if any: "Deviations from Plan" is NOT "None", "Issues Encountered" is NOT "None", file count >= 10
- COMPACT otherwise

Use visual patterns from @~/.claude/get-shit-done/references/ui-brand.md "Check-in Boxes" section:
- For wave check-ins: use wave check-in compact or expanded pattern
- For plan check-ins: use plan check-in compact or expanded pattern
- For failure check-ins: use failure check-in pattern

**Interaction loop:**

1. Present check-in display (compact or expanded, wave or plan format)

2. Use AskUserQuestion:
   - header: "{Type} Complete" (e.g., "Wave 2 Complete" or "Plan 02-01 Complete")
   - question: "{status summary line}"
   - options:
     - "Continue" -- Proceed to next wave/plan
     - "Review" -- See detailed summary or git diff
     - "Stop" -- Halt execution and save progress
   - allowsInput: true (enables freeform text as the "adjust" mechanism -- user types adjustment text into the Other/freeform field provided by AskUserQuestion)

   Note: "Adjust" is NOT a separate named option. It is available through AskUserQuestion's built-in "Other" freeform input field (allowsInput: true). The three NAMED options are Continue, Review, and Stop only.

3. Handle response:

   IF "Continue":
     Return to execute_waves. Proceed to next wave/plan.

   IF "Review":
     Ask follow-up: "Summary (SUMMARY.md content) or Diff (git diff since last check-in)?"
     - "Summary": Read and display SUMMARY.md content for each completed plan in scope
     - "Diff": Run `git diff {last_checkin_commit}..HEAD` where last_checkin_commit is tracked per check-in
     After displaying, RETURN TO STEP 2 (re-present options). Do NOT auto-continue.

   IF "Stop":
     Invoke /gsd:pause-work logic:
     - Record current position (which wave/plan was last completed, which remain)
     - Write .continue-here.md with full state
     - Present resume path: /gsd:resume-work
     EXIT execution entirely.

   IF freeform text (user typed into Other/adjust field):
     Write adjustment to STATE.md under Accumulated Context > Adjustments:
     ```
     - [{check-in label}]: "{user's text}"
     ```
     Confirm: "Adjustment noted for next agent: {summary}"
     RETURN TO STEP 2 (re-present options). Do NOT auto-continue.

**Tracking last commit for diff:**
Before each check-in, record current HEAD as `LAST_CHECKIN_COMMIT`:
```bash
LAST_CHECKIN_COMMIT=$(git rev-parse HEAD 2>/dev/null || echo "")
```
Update after each check-in's "Continue" response.
</step>

<step name="execute_waves">
Execute each wave in sequence. Within a wave: parallel if `PARALLELIZATION=true`, sequential if `false`.

**Plan granularity sequential override:**

IF CHECKIN_GRANULARITY is "plan" AND CURRENT_MODE is NOT "yolo":
  Plans within each wave execute ONE AT A TIME (sequential, not parallel).
  After each plan completes, invoke checkin_interaction with type="plan".
  This overrides the normal parallel spawning behavior for the wave.

IF CHECKIN_GRANULARITY is "wave" or "phase":
  Plans within each wave execute in PARALLEL (existing behavior unchanged).

**For each wave:**

1. **Describe what's being built (BEFORE spawning):**

   Read each plan's `<objective>`. Extract what's being built and why.

   ```
   ---
   ## Wave {N}

   **{Plan ID}: {Plan Name}**
   {2-3 sentences: what this builds, technical approach, why it matters}

   Spawning {count} agent(s)...
   ---
   ```

   - Bad: "Executing terrain generation plan"
   - Good: "Procedural terrain generator using Perlin noise — creates height maps, biome zones, and collision meshes. Required before vehicle physics can interact with ground."

2. **Read files and spawn agents:**

   Content must be inlined — `@` syntax doesn't work across Task() boundaries.

   ```bash
   PLAN_CONTENT=$(cat "{plan_path}")
   STATE_CONTENT=$(cat .planning/STATE.md)
   CONFIG_CONTENT=$(cat .planning/config.json 2>/dev/null)
   ```

   Each agent prompt:

   ```
   <objective>
   Execute plan {plan_number} of phase {phase_number}-{phase_name}.
   Commit each task atomically. Create SUMMARY.md. Update STATE.md.
   </objective>

   <execution_context>
   @~/.claude/get-shit-done/workflows/execute-plan.md
   @~/.claude/get-shit-done/templates/summary.md
   @~/.claude/get-shit-done/references/checkpoints.md
   @~/.claude/get-shit-done/references/tdd.md
   </execution_context>

   <context>
   Plan:
   {plan_content}

   Project state:
   {state_content}

   Config (if exists):
   {config_content}
   </context>

   <success_criteria>
   - [ ] All tasks executed
   - [ ] Each task committed individually
   - [ ] SUMMARY.md created in plan directory
   - [ ] STATE.md updated with position and decisions
   </success_criteria>
   ```

3. **Wait for all agents in wave to complete.**

4. **Report completion — spot-check claims first:**

   For each SUMMARY.md:
   - Verify first 2 files from `key-files.created` exist on disk
   - Check `git log --oneline --all --grep="{phase}-{plan}"` returns ≥1 commit
   - Check for `## Self-Check: FAILED` marker

   If ANY spot-check fails: report which plan failed, route to failure handler — ask "Retry plan?" or "Continue with remaining waves?"

   If pass:
   ```
   ---
   ## Wave {N} Complete

   **{Plan ID}: {Plan Name}**
   {What was built — from SUMMARY.md}
   {Notable deviations, if any}

   {If more waves: what this enables for next wave}
   ---
   ```

   - Bad: "Wave 2 complete. Proceeding to Wave 3."
   - Good: "Terrain system complete — 3 biome types, height-based texturing, physics collision meshes. Vehicle physics (Wave 3) can now reference ground surfaces."

3b. **Wave check-in (if configured):**

    IF CHECKIN_GRANULARITY is "wave" AND CURRENT_MODE is NOT "yolo":
      Use wave check-in pattern from @~/.claude/get-shit-done/references/ui-brand.md "Check-in Boxes" section (compact or expanded per adaptive density rule).
      Invoke checkin_interaction with:
      - type: "wave"
      - status: wave number, completed plan IDs, total files changed, next wave description
      - context: Wave {N}

      If checkin_interaction returns "stop", exit execution.
      If "continue", proceed to next wave.

    IF CHECKIN_GRANULARITY is "plan":
      Check-ins already happened after each plan (sequential override above). No additional wave check-in needed.
      (CONTEXT.md decision: "At plan granularity, all check-ins are plan-level -- no special treatment at wave boundaries")

    IF CHECKIN_GRANULARITY is "phase" or "none":
      No check-in. Proceed to next wave.

4. **Handle failures:**

   If any agent in wave fails:
   - Report which plan failed and why

   IF PAUSE_ON_FAILURE is "true" AND CURRENT_MODE is NOT "yolo":
     Use failure check-in pattern from @~/.claude/get-shit-done/references/ui-brand.md "Check-in Boxes" section.
     Invoke checkin_interaction with:
     - type: "failure"
     - status: failed plan ID, error reason, remaining plans in wave, remaining waves
     - context: Plan {ID} Failed

   ELSE (PAUSE_ON_FAILURE is false OR YOLO mode):
     Ask user (existing behavior): "Continue with remaining waves?" or "Stop execution?"
     If continue: proceed to next wave (dependent plans may also fail)
     If stop: exit with partial completion report

5. **Execute checkpoint plans between waves** — see `<checkpoint_handling>`.

6. **Proceed to next wave.**
</step>

<step name="checkpoint_handling">
Plans with `autonomous: false` require user interaction.

**Flow:**

1. Spawn agent for checkpoint plan
2. Agent runs until checkpoint task or auth gate → returns structured state
3. Agent return includes: completed tasks table, current task + blocker, checkpoint type/details, what's awaited
4. **Present to user:**
   ```
   ## Checkpoint: [Type]

   **Plan:** 03-03 Dashboard Layout
   **Progress:** 2/3 tasks complete

   [Checkpoint Details from agent return]
   [Awaiting section from agent return]
   ```
5. User responds: "approved"/"done" | issue description | decision selection
6. **Spawn continuation agent (NOT resume)** using continuation-prompt.md template:
   - `{completed_tasks_table}`: From checkpoint return
   - `{resume_task_number}` + `{resume_task_name}`: Current task
   - `{user_response}`: What user provided
   - `{resume_instructions}`: Based on checkpoint type
7. Continuation agent verifies previous commits, continues from resume point
8. Repeat until plan completes or user stops

**Why fresh agent, not resume:** Resume relies on internal serialization that breaks with parallel tool calls. Fresh agents with explicit state are more reliable.

**Checkpoints in parallel waves:** Agent pauses and returns while other parallel agents may complete. Present checkpoint, spawn continuation, wait for all before next wave.
</step>

<step name="aggregate_results">
After all waves:

```markdown
## Phase {X}: {Name} Execution Complete

**Waves:** {N} | **Plans:** {M}/{total} complete

| Wave | Plans | Status |
|------|-------|--------|
| 1 | plan-01, plan-02 | ✓ Complete |
| CP | plan-03 | ✓ Verified |
| 2 | plan-04 | ✓ Complete |

### Plan Details
1. **03-01**: [one-liner from SUMMARY.md]
2. **03-02**: [one-liner from SUMMARY.md]

### Issues Encountered
[Aggregate from SUMMARYs, or "None"]
```

**Phase completion gate:**

IF CURRENT_MODE is NOT "yolo" AND CHECKIN_GRANULARITY is NOT "none":
  Present phase completion summary.
  Invoke checkin_interaction with:
  - type: "phase"
  - status: total waves, total plans, aggregate files changed
  - context: Phase {X} Complete

  This gate fires at phase, wave, and plan granularity levels.
  CONTEXT.md decision: "At phase granularity (default), no mid-phase check-ins BUT always pause at phase completion before marking done -- gives user a final gate."

  If "stop", exit before verification.
  If "continue", proceed to verify_phase_goal.

When CHECKIN_GRANULARITY is "none": Skip phase completion gate entirely. Proceed directly to verify_phase_goal. (Verification still runs -- `none` skips check-ins, not verification.)
</step>

<step name="verify_phase_goal">
Verify phase achieved its GOAL, not just completed tasks.

```
Task(
  prompt="Verify phase {phase_number} goal achievement.
Phase directory: {phase_dir}
Phase goal: {goal from ROADMAP.md}
Check must_haves against actual codebase. Create VERIFICATION.md.",
  subagent_type="gsd-verifier",
  model="{verifier_model}"
)
```

Read status:
```bash
grep "^status:" "$PHASE_DIR"/*-VERIFICATION.md | cut -d: -f2 | tr -d ' '
```

| Status | Action |
|--------|--------|
| `passed` | → update_roadmap |
| `human_needed` | Present items for human testing, get approval or feedback |
| `gaps_found` | Present gap summary, offer `/gsd:plan-phase {phase} --gaps` |

**If human_needed:**
```
## ✓ Phase {X}: {Name} — Human Verification Required

All automated checks passed. {N} items need human testing:

{From VERIFICATION.md human_verification section}

"approved" → continue | Report issues → gap closure
```

**If gaps_found:**
```
## ⚠ Phase {X}: {Name} — Gaps Found

**Score:** {N}/{M} must-haves verified
**Report:** {phase_dir}/{phase}-VERIFICATION.md

### What's Missing
{Gap summaries from VERIFICATION.md}

---
## ▶ Next Up

`/gsd:plan-phase {X} --gaps`

<sub>`/clear` first → fresh context window</sub>

Also: `cat {phase_dir}/{phase}-VERIFICATION.md` — full report
Also: `/gsd:verify-work {X}` — manual testing first
```

Gap closure cycle: `/gsd:plan-phase {X} --gaps` reads VERIFICATION.md → creates gap plans with `gap_closure: true` → user runs `/gsd:execute-phase {X} --gaps-only` → verifier re-runs.
</step>

<step name="update_roadmap">
Mark phase complete in ROADMAP.md (date, status).

```bash
node ~/.claude/get-shit-done/bin/gsd-tools.js commit "docs(phase-{X}): complete phase execution" --files .planning/ROADMAP.md .planning/STATE.md .planning/phases/{phase_dir}/*-VERIFICATION.md .planning/REQUIREMENTS.md
```
</step>

<step name="offer_next">

**If more phases:**
```
## Next Up

**Phase {X+1}: {Name}** — {Goal}

`/gsd:plan-phase {X+1}`

<sub>`/clear` first for fresh context</sub>
```

**If milestone complete:**
```
MILESTONE COMPLETE!

All {N} phases executed.

`/gsd:complete-milestone`
```
</step>

</process>

<context_efficiency>
Orchestrator: ~10-15% context. Subagents: fresh 200k each. No polling (Task blocks). No context bleed.
</context_efficiency>

<failure_handling>
- **Agent fails mid-plan:** Missing SUMMARY.md → report, ask user how to proceed
- **Dependency chain breaks:** Wave 1 fails → Wave 2 dependents likely fail → user chooses attempt or skip
- **All agents in wave fail:** Systemic issue → stop, report for investigation
- **Checkpoint unresolvable:** "Skip this plan?" or "Abort phase execution?" → record partial progress in STATE.md
</failure_handling>

<resumption>
Re-run `/gsd:execute-phase {phase}` → discover_plans finds completed SUMMARYs → skips them → resumes from first incomplete plan → continues wave execution.

STATE.md tracks: last completed plan, current wave, pending checkpoints.
</resumption>
