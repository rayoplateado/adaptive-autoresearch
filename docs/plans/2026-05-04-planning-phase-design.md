# Planning Phase Design

Add a planning phase to adaptive-autoresearch that analyzes detected issues,
groups them by theme and dependency, and enables parallel subagent execution
via Claude Code's Agent tool and worktree isolation.

## Motivation

The current loop picks the weakest metric each iteration and fixes issues
one at a time, serially. This misses three opportunities:

1. **Batching** — related issues (all auth checks, all type fixes) are more
   efficient to fix together than interleaved with unrelated work.
2. **Ordering** — fixing a base type can auto-resolve downstream `any` casts.
   Without a plan, the loop doesn't know this.
3. **Parallelism** — independent issue groups can run as parallel subagents
   in worktrees, cutting wall-clock time proportionally.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Where in the pipeline | New Phase 8 (between baseline and loop) | Clean separation, own checkpoint, easy to skip/re-run |
| Parallel execution model | Worktree isolation + sequential merge | Safe, uses Claude Code primitives, predictable merge |
| Task granularity | Two-tier (group tasks with issue lists in description) | Clean task list (5-8 items) with drill-down detail |
| Re-evaluation cadence | After each group merge + when stuck | Adaptive without per-iteration overhead |

## New Phase 8: Plan

Runs once after baseline (Phase 7), before the loop (now Phase 9).

### Step 1: Analyze findings

For each metric above target, enumerate individual occurrences with
file:line locations. Example: `typescript-any-count` = 38 means list
all 38 locations.

### Step 2: Group by theme and dependency

Cluster issues into groups based on:

- **Domain affinity** — all auth issues together, all type issues together
- **File overlap** — issues touching the same files go in the same group
- **Dependency ordering** — if fixing issue X auto-resolves issue Y, same group

### Step 3: Order groups

Priority:

1. Groups with downstream unblocking effect
2. User-stated priority (from Phase 5 checkpoint)
3. Highest issue count (biggest impact)
4. Source skill authority (more installs = more weight)

### Step 4: Identify parallelism

Groups with zero file overlap can run as parallel subagents. Groups with
file overlap must run sequentially. Build a dependency graph between groups.

### Step 5: Create task list

One TaskCreate per group (two-tier: group name + issue list in description).
Set `addBlockedBy` for dependent groups.

### Step 6: Present to user

Show groups, ordering, parallelism opportunities, estimated iterations per
group. Third user checkpoint (after fitness profile and instrumentation).

## Modified Phase 9: Plan-Driven Loop

The loop executes the plan instead of "pick weakest metric."

### Execution model

```
1. Read the plan (group ordering + dependency graph)
2. Identify next runnable groups (no blockers)
3. Independent groups → spawn parallel subagents (isolation: worktree)
   Single/dependent groups → run in main context
4. Each subagent runs a mini-loop within its group:
   - Pick next issue from the group's list
   - Fix → verify (build/tests) → measure → keep/discard
   - Update task description with progress ("5/12 fixed")
   - Repeat until group exhausted or stuck
5. When subagent finishes:
   - Merge worktree into main branch
   - Re-run full metrics (run-all.sh) for cross-group regressions
   - If regression → revert merge, flag group for sequential retry
   - If clean → mark group completed, unblock dependent groups
6. Every N groups (~10 iterations), re-evaluate:
   - Re-enumerate remaining issues (some auto-resolved)
   - Update task descriptions with new counts
   - Drop groups now at target
7. Continue until all groups done or user stops
```

### What each subagent receives

- The fitness profile (fitness.yaml)
- Relevant skill content for its domain (just-in-time loading)
- Its group's issue list (specific files and locations)
- The metric scripts for validating its own progress
- Instructions to commit each kept change independently

### Subagent constraints

- Must not modify metrics outside its group's scope
- Must not touch files outside its group's file set
- Must not modify fitness.yaml or metric scripts
- Cannot spawn further subagents (Claude Code restriction)

### Merge protocol

1. Subagent finishes → parent receives summary
2. Parent merges worktree branch into main
3. Parent runs `run-all.sh` on merged state
4. Regression → `git merge --abort`, queue for sequential retry
5. Clean → commit merge, update session.jsonl

## Re-evaluation Checkpoints

Triggered after each group merge, after `max_no_improvement` consecutive
discards, or when the user resumes.

What happens:

- Run `run-all.sh`, re-enumerate remaining group issues
- Update task descriptions with new counts
- Mark auto-resolved groups as completed
- Adjust ordering if new dependencies emerged
- Flag stuck groups for user attention

What does NOT happen:

- No re-discovery or re-fetch of skills
- No regeneration of metric scripts
- No changes to fitness.yaml targets
- No re-baselining

## Session Persistence

### New file: `.adaptive-autoresearch/plan.yaml`

```yaml
created_at: <timestamp>
last_evaluated_at: <timestamp>

groups:
  - name: auth-coverage
    domain: security
    metric: auth-coverage
    source_skill: "supercent-io/skills-template/security-best-practices"
    status: pending  # pending | in_progress | completed | stuck
    issues:
      - file: convex/domain/bots/mutations.ts
        line: 45
        description: "createBot mutation missing auth check"
        status: pending  # pending | fixed | wont_fix
    progress: "0/8 fixed"
    parallel: true
    blocked_by: []

execution:
  parallel_waves:
    - [auth-coverage, type-safety]
    - [query-projections]
  max_parallel_agents: 3
  re_evaluate_after_groups: 1
```

### Updated directory structure

```
.adaptive-autoresearch/
├── fitness.yaml
├── plan.yaml             # NEW
├── session.md
├── session.jsonl
├── run-all.sh
└── metrics/
```

### Resume logic

A fresh agent reads `plan.yaml` alongside `session.md`. If groups are
`in_progress`, check for orphaned worktree branches, re-run metrics,
continue from where the plan left off.

## Changes to SKILL.md

- Phases 0-7: no changes
- New Phase 8: Plan (this design)
- Phase 9 (was Phase 8): Plan-Driven Loop with subagent support
- Session Persistence: add plan.yaml documentation
- Resuming: add plan-aware resume logic
- Overview diagram: add PLAN box between BASELINE and LOOP

The skill does not mandate subagents — projects with 1-2 issue groups
run sequentially in main context. Parallelism is opportunistic. No new
dependencies; everything uses built-in Claude Code primitives.
