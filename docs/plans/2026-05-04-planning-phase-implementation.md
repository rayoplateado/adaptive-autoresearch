# Planning Phase Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a planning phase (Phase 8) and modify the loop (Phase 9) in SKILL.md to support issue grouping, dependency ordering, parallel subagent execution, and two-tier task tracking.

**Architecture:** Single file modification (SKILL.md) with surgical edits: update overview diagram, insert new Phase 8 section, rewrite Phase 8 → Phase 9 with subagent support, update session persistence and resume logic. Design doc at `docs/plans/2026-05-04-planning-phase-design.md` is the source of truth.

**Tech Stack:** Markdown (SKILL.md), YAML examples (plan.yaml schema)

---

### Task 1: Update the overview diagram

**Files:**
- Modify: `SKILL.md:30-47`

**Step 1: Edit the ASCII diagram**

Replace the current diagram block (lines 30-47) with an updated version that adds PLAN between BASELINE and LOOP, and renumbers phases:

```markdown
```
┌──────────────────────────────────────────────────────────────┐
│  0. PREREQS    — Git clean, build passes, skills CLI check   │
│  1. DETECT     — What is this project? Stack, type, deps     │
│  2. DISCOVER   — Search skills.sh for relevant expertise     │
│  3. FETCH      — Read the SKILL.md content from GitHub       │
│  4. EXTRACT    — Pull quality criteria, rules, patterns out  │
│  5. COMPOSE    — Build the fitness profile from all sources   │
│  6. INSTRUMENT — Generate audit scripts, validate them       │
│  7. BASELINE   — Run scripts on UNCHANGED code = iteration 0 │
│  8. PLAN       — Group issues, order, identify parallelism   │
│  9. LOOP       — Change → measure → keep/discard → repeat    │
└──────────────────────────────────────────────────────────────┘
```

Steps 0-7 happen once, before any code changes.
Step 8 creates the execution plan from baseline results.
Step 9 runs until all targets are met or the user stops it.
```

**Step 2: Verify the edit**

Read `SKILL.md:28-50` to confirm the diagram is correct and aligned.

**Step 3: Commit**

```bash
git add SKILL.md
git commit -m "update overview diagram: add PLAN phase, renumber LOOP to 9"
```

---

### Task 2: Insert new Phase 8: Plan

**Files:**
- Modify: `SKILL.md` — insert after the Phase 7 section (after line 576, before the current Phase 8 heading)

**Step 1: Write the Phase 8 section**

Insert the following after the `---` that ends Phase 7 (line 577) and before the current Phase 8 heading:

```markdown
## Phase 8: Plan

After the baseline captures metrics, analyze the results and build an
execution plan before making any code changes. This phase groups issues,
identifies dependencies, and enables parallel execution.

### Step 1: Analyze findings

For each metric above its target, enumerate individual occurrences with
file:line locations. Use the metric scripts and grep/AST analysis to build
a concrete list — not estimates, actual locations.

Example: if `typescript-any-count` = 38, list all 38 occurrences with
file paths and line numbers. If `auth-coverage` = 16, list the 16
unprotected endpoints.

### Step 2: Group by theme and dependency

Cluster issues into groups based on:

- **Domain affinity** — all auth issues together, all type issues together
- **File overlap** — issues touching the same files go in the same group
- **Dependency ordering** — if fixing issue X auto-resolves issue Y
  (e.g., fixing a base type eliminates downstream `any` casts), they
  belong in the same group

Aim for 3-8 groups. Fewer than 3 means the project is simple enough to
not need this phase — skip to the loop. More than 8 means groups are too
granular — merge related ones.

### Step 3: Order groups

Prioritize by:

1. **Downstream unblocking** — groups whose fixes make other groups cheaper
2. **User-stated priority** — from the Phase 5 checkpoint
3. **Issue count** — groups with more issues have more impact
4. **Source authority** — skills with more installs carry more weight

### Step 4: Identify parallelism

Groups with zero file overlap can run as parallel subagents in worktrees.
Groups with file overlap must run sequentially. Build a dependency graph:

```
group-auth ──┐
             ├──→ group-query-projections (shares files with both)
group-types ─┘
```

Mark each group as `parallel: true` or `parallel: false` with its
`blocked_by` list.

### Step 5: Create task list

Create one task per group using TaskCreate. Each task uses two-tier
tracking:

- **Subject**: group name with issue count (e.g., "Fix auth coverage (12 issues)")
- **Description**: full issue list with file:line locations

For groups with dependencies, set `addBlockedBy` so blocked groups don't
start until their prerequisites complete.

### Step 6: Persist the plan

Write `.adaptive-autoresearch/plan.yaml`:

```yaml
created_at: <timestamp>
last_evaluated_at: <timestamp>

groups:
  - name: <group-name>
    domain: <security|types|performance|...>
    metric: <metric-name>
    source_skill: "<skill-identifier>"
    status: pending  # pending | in_progress | completed | stuck
    issues:
      - file: <path>
        line: <number>
        description: "<what's wrong>"
        status: pending  # pending | fixed | wont_fix
      # ... all issues in this group
    progress: "0/N fixed"
    parallel: <true|false>
    blocked_by: [<group-names>]

execution:
  parallel_waves:
    - [<groups that can run simultaneously>]  # wave 1
    - [<groups that depend on wave 1>]        # wave 2
    # ...
  max_parallel_agents: 3
  re_evaluate_after_groups: 1
```

### Step 7: Present to user

Show the plan:
- Groups with their issue counts and ordering
- Dependency graph (which groups block which)
- Parallel waves (what runs simultaneously)
- Estimated total iterations

Ask: "Does this plan look right? Anything to reorder, merge, or split?"

This is the third user checkpoint (after fitness profile in Phase 5 and
instrumentation in Phase 6).
```

**Step 2: Verify the insertion**

Read the area around the insertion point to confirm it flows correctly
from Phase 7 into Phase 8 into Phase 9.

**Step 3: Commit**

```bash
git add SKILL.md
git commit -m "add Phase 8: Plan — issue grouping, ordering, parallelism"
```

---

### Task 3: Rewrite Phase 8 → Phase 9: Plan-Driven Loop

**Files:**
- Modify: `SKILL.md` — replace the current "Phase 8: The Autonomous Loop" section

**Step 1: Replace the Phase 8 heading and intro**

Change `## Phase 8: The Autonomous Loop` to `## Phase 9: The Plan-Driven Loop`
and update the intro paragraph to reference the plan.

**Step 2: Replace the iteration steps**

Replace the current "Each iteration" block with the plan-driven execution
model:

```markdown
### Execution model

The loop executes the plan from Phase 8 instead of picking the weakest
metric each iteration.

```
1. READ PLAN     — Load plan.yaml, identify next runnable groups
2. SPAWN         — Independent groups → parallel subagents (worktree)
                   Dependent/single groups → run in main context
3. MINI-LOOP     — Each agent runs within its group:
                   a. Pick next issue from the group's list
                   b. FOCUS — Load relevant skill content for the domain
                   c. IMPLEMENT — Fix the issue
                   d. VERIFY — Constraints pass? If fail → revert, next issue
                   e. MEASURE — Run metric scripts
                   f. EVALUATE — Improved? Keep (commit). Not? Discard (revert)
                   g. Update task description with progress ("5/12 fixed")
                   h. JOURNAL — Append to session.jsonl
                   i. Next issue in group
4. MERGE         — Subagent done → merge worktree into main
5. VALIDATE      — Run full run-all.sh on merged state
                   Regression? → revert merge, retry group sequentially
                   Clean? → mark group completed, unblock dependents
6. RE-EVALUATE   — After each merge (or ~10 iterations):
                   - Re-enumerate remaining issues
                   - Update task descriptions
                   - Drop groups now at target
                   - Flag stuck groups for user attention
7. NEXT WAVE     — Identify newly unblocked groups, go to step 2
```

If the project has only 1-2 issue groups, skip subagent spawning and run
sequentially in main context — parallelism is opportunistic, not mandatory.
```

**Step 3: Add the subagent specification subsection**

After the execution model, add:

```markdown
### What each subagent receives

The parent agent spawns subagents using the Agent tool with
`isolation: worktree`. Each subagent's prompt includes:

- The fitness profile (fitness.yaml) for context
- The relevant skill content for the group's domain
- The group's issue list with specific file:line locations
- The metric scripts needed to validate progress
- Instructions to commit each kept change as a separate commit

### Subagent constraints

- Must not modify metrics outside its group's scope
- Must not touch files outside its group's file set (unless the plan
  explicitly marks shared files)
- Must not modify fitness.yaml or metric scripts
- Cannot spawn further subagents (Claude Code does not allow nesting)

### Merge protocol

1. Subagent finishes → parent receives a summary of changes and
   metric deltas within the group
2. Parent merges the worktree branch into main
3. Parent runs `run-all.sh` on the merged state
4. If any metric regressed vs pre-merge state → revert the merge,
   log the conflict, queue the group for sequential retry in main context
5. If clean → accept the merge, update session.jsonl with the group's
   full results, mark the group task as completed
```

**Step 4: Keep existing subsections that still apply**

The following subsections remain unchanged (just renumber references):
- "The MEASURE step is sacred"
- "The FOCUS step is key" (now happens inside the mini-loop)
- "Keep/discard logic"
- "What the agent should NOT do"

**Step 5: Add re-evaluation subsection**

```markdown
### Re-evaluation checkpoints

After each group merge, after `max_no_improvement` consecutive discards
within a group, or when the user resumes after an interruption:

1. Run `run-all.sh` to get current state
2. Re-enumerate issues for remaining groups (some may have auto-resolved)
3. Update task descriptions with new counts
4. Mark auto-resolved groups as completed
5. If new dependencies emerged, adjust group ordering
6. Flag stuck groups for user attention

Re-evaluation does NOT:
- Re-discover or re-fetch skills
- Regenerate metric scripts
- Change fitness.yaml targets
- Re-baseline (the original baseline remains the canonical "before")
```

**Step 6: Verify the full section**

Read the entire Phase 9 section to confirm it's coherent and all
subsections flow correctly.

**Step 7: Commit**

```bash
git add SKILL.md
git commit -m "rewrite Phase 8 → Phase 9: plan-driven loop with subagents"
```

---

### Task 4: Update Session Persistence section

**Files:**
- Modify: `SKILL.md` — the "Session Persistence" section

**Step 1: Add plan.yaml to the directory structure**

Update the directory listing to include `plan.yaml`:

```
.adaptive-autoresearch/
├── fitness.yaml          # fitness profile (metrics, targets, sources)
├── plan.yaml             # execution plan (groups, ordering, parallelism)
├── session.md            # living session document
├── session.jsonl         # append-only iteration log
├── run-all.sh            # orchestrator
└── metrics/              # one script per metric
```

**Step 2: Add plan.yaml documentation**

After the `session.md` subsection, add:

```markdown
### `plan.yaml`

Execution plan generated in Phase 8. Contains:
- Issue groups with file:line locations and status tracking
- Dependency graph between groups
- Parallel execution waves
- Progress counters per group

Updated during re-evaluation checkpoints. A fresh agent reads this to
understand what work remains and which groups can run next.
```

**Step 3: Update session.md description**

Add "Plan summary and group progress" to the list of what session.md
includes.

**Step 4: Commit**

```bash
git add SKILL.md
git commit -m "update session persistence: add plan.yaml documentation"
```

---

### Task 5: Update Resuming a Session section

**Files:**
- Modify: `SKILL.md` — the "Resuming a session" section

**Step 1: Add plan-aware resume logic**

Replace the current resume steps with:

```markdown
## Resuming a session

If `.adaptive-autoresearch/session.md` exists:

1. Read `session.md` — it has everything: project context, skill sources,
   metrics, current state, history
2. Read `session.jsonl` for recent iterations
3. Read `fitness.yaml` for the full profile
4. Read `plan.yaml` for the execution plan and group progress
5. **Run `run-all.sh`** to verify current state matches journal
6. If numbers match: continue from where the plan left off
7. If numbers diverge: log the discrepancy, re-baseline, then continue
8. If groups are `in_progress`: check for orphaned worktree branches
   (from interrupted subagents). If found, evaluate their changes and
   either merge or discard before continuing.

Do NOT re-discover, re-fetch, or re-compose unless the user explicitly
asks to refresh the knowledge sources. Do NOT regenerate scripts unless
one is broken or the user requests it. Do NOT re-plan unless the user
asks — the existing plan.yaml is the source of truth.
```

**Step 2: Commit**

```bash
git add SKILL.md
git commit -m "update resume logic: plan-aware with orphaned worktree handling"
```

---

### Task 6: Update Communication section

**Files:**
- Modify: `SKILL.md` — the "Communicating with the user" section

**Step 1: Add plan checkpoint to "At the start"**

After the existing items in "At the start", add the plan presentation
as the third checkpoint.

**Step 2: Update "During the loop"**

Add to the existing items:
- Group completion notifications
- Parallel wave progress (e.g., "Wave 1: 2/3 groups done")
- Merge success/failure notifications

**Step 3: Commit**

```bash
git add SKILL.md
git commit -m "update communication section: add plan checkpoint and group progress"
```

---

### Task 7: Update the intro paragraph

**Files:**
- Modify: `SKILL.md:16-24`

**Step 1: Update the description**

The current intro says "The loop is simple: analyze → plan → change →
verify → measure → keep or discard → repeat." Update to reflect the
new planning phase:

```markdown
The loop is simple: analyze → plan → group → change → verify → measure →
keep or discard → repeat. What makes it powerful is that the quality
criteria come from the collective knowledge of thousands of community
skills, fetched on demand, not hardcoded — and independent issue groups
can run in parallel as subagents for faster execution.
```

**Step 2: Commit**

```bash
git add SKILL.md
git commit -m "update intro: reflect planning phase and parallel execution"
```

---

### Task 8: Final review pass

**Step 1: Read the full SKILL.md**

Read the entire file end-to-end. Check:
- Phase numbering is consistent (0-9, no gaps, no duplicates)
- All cross-references point to correct phase numbers
- The overview diagram matches the phase headings
- plan.yaml is mentioned in all relevant sections
- No orphaned references to "Phase 8" meaning the old loop

**Step 2: Fix any inconsistencies found**

**Step 3: Commit any fixes**

```bash
git add SKILL.md
git commit -m "fix: consistency pass on phase numbering and cross-references"
```
