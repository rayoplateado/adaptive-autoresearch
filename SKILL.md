---
name: adaptive-autoresearch
description: >
  Autonomous code improvement loop that discovers what "excellent" means
  for any project by reading community skills from skills.sh, then iterates
  toward that standard automatically. Use when the user says "optimize this
  project", "polish this code", "autodev", "make this production-ready",
  "I vibecoded this, clean it up", or wants autonomous iterative improvement.
  Also use when building something new to high standards. The skill has zero
  built-in domain knowledge — it discovers relevant expertise from the
  skills.sh ecosystem at runtime, reads it, extracts quality criteria, and
  uses it to drive a plan-and-execute loop with parallel subagent support,
  inspired by Karpathy's autoresearch. Works with any stack, any language,
  any project type.
---

# adaptive-autoresearch

An autonomous improvement loop with no built-in opinions. It discovers what
"excellent" means for YOUR project by reading what the community already
knows, then iterates until it gets there.

The loop is simple: analyze → plan → group → change → verify → measure →
keep or discard → repeat. What makes it powerful is that the quality
criteria come from the collective knowledge of thousands of community
skills, fetched on demand, not hardcoded — and independent issue groups
can run in parallel as subagents for faster execution.

---

## How it works — overview

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

Phases 0-7 happen once, before any code changes.
Phase 8 creates the execution plan from baseline results.
Phase 9 runs until all targets are met or the user stops it.

---

## Safety Layer

A cross-cutting safety system that validates skills and scripts before they
can influence code changes or execute on your machine. On by default.

### Configuration: `.adaptive-autoresearch/safety.yaml`

Generated automatically in Phase 0 with these defaults:

```yaml
# .adaptive-autoresearch/safety.yaml
enabled: true

trust:
  # Skills below BOTH thresholds trigger a user warning
  min_installs: 5000
  min_age_days: 30

scripts:
  # Patterns that flag a generated metric script for user review
  blocked_patterns:
    - rm\b
    - curl\b
    - wget\b
    - eval\b
    - exec\b
    - ">>"
    - ">"
    - sudo\b
    - chmod\b
    - kill\b
    - mkfifo\b
    - nc\b
    - python.*-c.*open\(.*,\s*['"]w
  # Writes to these paths are allowed and don't trigger warnings
  allowed_write_paths:
    - .adaptive-autoresearch/
    - /tmp/

overrides:
  # Populated by user approvals during the run
  approved_skills: []
  approved_scripts: []
```

Trust thresholds use AND — both must be below to trigger a warning.
Blocked patterns are regexes applied line-by-line to generated scripts.
Approved scripts are tracked by checksum — re-flagged if content changes.

### Gate 1: Skill Trust Validation

Applied in Phase 3 (Fetch), before reading a skill's SKILL.md content.

For each discovered skill:

1. Collect: publisher, install count, publish date (or first commit date)
2. Check against `trust.min_installs` AND `trust.min_age_days`
3. Meets thresholds → proceed silently, log `trust: auto`
4. Below thresholds → warn the user:

```
⚠ Low-trust skill detected:
  Name: some-user/obscure-repo/security-tips
  Installs: 320
  Age: 12 days

  Use this skill? [y/n/details]
```

5. "details" shows the first 20 lines of the SKILL.md
6. User approves → add to `overrides.approved_skills`, proceed
7. User rejects → skip this skill, note gap in session.md

No hard blocks. Every skill can be approved by the user.

### Gate 2: Script Safety Review

Applied in Phase 6 (Instrument), after generating each metric script but
before first execution.

For each generated script:

1. Read line by line, match against `scripts.blocked_patterns`
2. Exclude matches targeting paths in `allowed_write_paths`
3. No matches → proceed, mark `review: clean`
4. Matches found → show the user:

```
⚠ Script review: metric-api-response-check.sh
  Line 7: curl -s "$ENDPOINT" | jq '.status'
  Line 12: echo "$result" > /tmp/api-check.txt

  Matched patterns: curl, >

  Approve this script? [y/n/edit]
```

5. User approves → add to `overrides.approved_scripts` with checksum
6. User rejects → mark metric as `manual: true`, skip automation
7. User chooses "edit" → re-run safety check on edited version

Runs once per script. Not re-flagged unless checksum changes.

### Non-interactive fallback

If the agent cannot prompt the user (CI, background mode), below-threshold
skills and flagged scripts are rejected by default. Only auto-approved
content runs. Better to have reduced coverage than unreviewed execution.

### Overrides are shareable

Since `safety.yaml` lives in `.adaptive-autoresearch/`, it can be committed.
Team members share trust decisions — one approval benefits everyone.

---

## Phase 0: Prerequisites

Before starting, verify:

1. **`skills` CLI is available** — run `npx skills --help` or `skills --help`.
   If not installed, the agent can still work but must skip Phase 2-3
   (Discover/Fetch) and rely only on locally installed skills and its
   own knowledge. Note this limitation in the session log.

2. **Git is clean** — `git status` should show no uncommitted changes.
   The loop makes commits and may need to revert. Dirty state = broken loop.

3. **Build and tests pass** — verify before measuring anything.
   If they don't pass, fix them first (this is iteration -1, not the loop).

---

## Phase 1: Detect

Scan the project. Identify:

- **Type**: api, frontend, cli, bot, library, fullstack, mobile
- **Language**: TypeScript, Python, Go, Rust, etc.
- **Framework**: Next.js, NestJS, FastAPI, Express, Svelte, etc.
- **UI library** (if frontend): shadcn, MUI, Chakra, Radix, etc.
- **CSS approach**: Tailwind, CSS modules, styled-components, vanilla
- **Test framework**: Jest, Vitest, pytest, Playwright, etc.
- **Package manager**: npm, pnpm, yarn, pip, cargo, etc.
- **Existing quality tools**: ESLint, Prettier, Semgrep, Storybook, etc.

Detection is done by reading package.json, pyproject.toml, Cargo.toml,
tsconfig, and scanning the file tree. No guessing — read the actual files.

Save the detection results. They drive everything that follows.

---

## Phase 2: Discover

Search for skills relevant to this project's stack using the `skills` CLI.
This is the key innovation: the agent has no opinions of its own — it finds
the best available community expertise.

### How to search

Use the `skills` CLI to discover relevant skills. Run multiple searches
to cover the detected stack:

```bash
# Search by framework / stack keywords
skills find next.js
skills find react
skills find tailwind
skills find convex

# Search by quality domain
skills find security
skills find testing
skills find accessibility
skills find performance

# Search by architecture concern
skills find "api design"
skills find "design system"
```

Each `skills find <keyword>` returns matching skills with names, publishers,
and install counts. Use this to build a candidate list.

To inspect what skills a known repository offers without installing:

```bash
skills add vercel-labs/agent-skills -l    # list skills in repo
skills add shadcn/ui -l                    # list skills in repo
skills add supercent-io/skills-template -l
```

### Also check locally installed skills

Before fetching anything remote, check what's already installed:

```bash
skills list              # project-level skills
skills list -g           # global skills
skills ls --json         # machine-readable for scripting
```

Already-installed skills are available at their local paths — no need to
fetch them from GitHub. Read them directly.

### What to look for

Prioritize skills with:
- High install counts (community-validated)
- From known publishers (vercel-labs, anthropics, shadcn, expo, etc.)
- Relevant to the detected stack
- Actionable content (rules, patterns, checks — not just philosophy)

### Aim for coverage across domains

Don't just find 5 performance skills. Find skills across these domains:

- **Stack best practices** (framework-specific patterns and anti-patterns)
- **Design system / UI quality** (if frontend)
- **Product completeness / UX** (if frontend)
- **Security** (for any project with a network surface)
- **Testing quality** (for any project with tests, or that should have them)
- **Performance** (for any project where speed matters)
- **Accessibility** (if frontend)
- **Code architecture** (for any project)

The goal is a well-rounded quality profile, not depth in one area.

---

## Phase 3: Fetch

For each relevant skill found, get its SKILL.md content. Prefer local
paths when available; fall back to GitHub raw URLs for remote-only skills.

### Strategy: local first, remote second

**Already installed skills** (found via `skills list` or `skills ls --json`):
- Read directly from their local path — no network needed
- Check both project-level and global paths

**Not yet installed — two options**:

1. **Install temporarily** (preferred for high-value skills):
   ```bash
   skills add owner/repo -s skill-name -g -y   # install globally, no prompt
   ```
   Then read from the local path. This also makes the skill available for
   future sessions.

2. **Fetch raw from GitHub** (for quick inspection without installing):
   Given `owner/repo/skill-name`, try these URLs in order:
   ```
   https://raw.githubusercontent.com/{owner}/{repo}/main/skills/{skill-name}/SKILL.md
   https://raw.githubusercontent.com/{owner}/{repo}/main/{skill-name}/SKILL.md
   https://raw.githubusercontent.com/{owner}/{repo}/main/.claude/skills/{skill-name}/SKILL.md
   ```

Fetch only the SKILL.md — it contains the core knowledge. If the skill
references files in `references/` that seem critical, fetch those too, but
be selective. Most of the value is in the SKILL.md itself.

### Context management

Don't dump all skills into context at once. Keep them in working memory
and load them selectively:

- During **profile composition**: skim all skills to extract metrics/rules
- During **the loop**: load only the skill relevant to the current
  improvement focus (working on security? load the security skill content)

This keeps context focused and avoids overwhelming the working set.

---

## Phase 4: Extract

Read each fetched skill and extract structured quality criteria. This is
where the agent's intelligence matters — it's reading natural language
instructions and turning them into measurable quality signals.

### What to extract from each skill

**Hard rules** — things that should NEVER appear or ALWAYS be present:
- "Never use inline styles" → metric: inline_styles_count, target: 0
- "Always use TypeScript strict mode" → check: tsconfig strict = true
- "Never use any type" → metric: any_type_count, target: 0
- "All API routes must validate input" → check per route

**Quality patterns** — things that constitute good practice:
- "Components should be small and focused" → metric: max_component_lines
- "Use design tokens instead of raw values" → metric: hardcoded_values_count
- "Every list view should have search, pagination, and sorting"
  → product completeness checks

**Anti-patterns** — things to flag and fix:
- "Avoid prop drilling" → metric: prop_drilling_depth
- "Don't use index as key in lists" → grep-based check
- "Avoid useEffect for derived state" → pattern detection

**Architecture guidance** — structural patterns to enforce:
- "Organize by feature, not by type" → directory structure check
- "Use server components by default" (Next.js) → pattern check
- "Separate data fetching from presentation" → component analysis

### Extraction output

For each skill, produce a structured summary:

```yaml
source: "vercel-labs/agent-skills/vercel-react-best-practices"
installs: 228000
metrics_extracted:
  - name: server_components_ratio
    check: "% of components that are server components"
    target: 0.70
    direction: higher
    scriptable: true  # can be measured by grep/count
  - name: use_client_count
    check: "Number of unnecessary 'use client' directives"
    target: 0
    direction: lower
    scriptable: true
rules_extracted:
  - "Prefer server components; add 'use client' only when needed"
  - "Use Next.js Image component, never raw <img>"
  - "Colocate data fetching with the component that needs it"
anti_patterns:
  - "useEffect for data fetching (use server components instead)"
  - "Client-side redirect (use middleware instead)"
strategies:
  - "Convert client components to server components where possible"
  - "Move data fetching to server components"
  - "Replace <img> with next/image"
```

---

## Phase 5: Compose

Aggregate all extracted knowledge into a single fitness profile for this
project. This is the configuration that drives the entire loop.

### Build `.adaptive-autoresearch/fitness.yaml`

```yaml
project:
  name: <detected>
  type: <detected>
  stack: <detected>
  detected_at: <timestamp>

sources:
  # Track which skills contributed to this profile
  - skill: "vercel-labs/agent-skills/vercel-react-best-practices"
    installs: 228000
    contributed: [server_components_ratio, use_client_count, ...]
  - skill: "shadcn/ui/shadcn"
    installs: 28000
    contributed: [ui_library_coverage, ...]
  - skill: "supercent-io/skills-template/security-best-practices"
    installs: 13000
    contributed: [vulnerability_score, ...]

loop:
  budget_seconds: 300
  max_iterations: 50
  max_no_improvement: 10

metrics:
  # One entry per metric. Each must have a script or be marked manual.
  - name: <metric_name>
    source: <which skill suggested this>
    script: .adaptive-autoresearch/metrics/metric-<name>.sh
    target: <number to reach>
    direction: <lower|higher>
  # Metrics that can't be scripted:
  - name: <metric_name>
    source: <which skill>
    manual: true
    manual_check: <how to verify manually>
    target: <criteria>

rules:
  # Hard rules aggregated from all skills
  - rule: "Never use inline styles"
    source: "vercel-labs/agent-skills/web-design-guidelines"
  # ...

strategies:
  # Improvement strategies grouped by domain
  performance:
    - strategy: "..."
      source: "..."
  security:
    - strategy: "..."
      source: "..."
  # ...

constraints:
  - name: existing_tests_pass
    command: "<detected>"
  - name: build_succeeds
    command: "<detected>"

guardrails:
  - "Changes must not remove user-facing functionality"
  - "Metric improvement must not game checks (no assert-true tests)"
  - "Component refactoring must preserve visual output"
```

### Scoring

Don't invent composite scores. Use the raw numbers from the scripts.

The fitness of a project is the set of metric values, not a single number.
When comparing before/after, show the table:

```
Metric                          before   after    Δ
typescript-any-count               38       0   -38 ✅
public-functions-without-auth      16       0   -16 ✅
tsc-errors                         55       0   -55 ✅
TOTAL ISSUES                      109       0  -100%
```

If the user needs a single number, use **total issues** (sum of all
metrics where direction = lower) and **reduction percentage**. This is
honest and verifiable — anyone can run the scripts and get the same
numbers.

### Prioritization

When deciding which metrics to tackle first, consider:
- **User priority**: if the user says "I care most about security",
  start there
- **Source authority**: skills with more installs carry more weight
- **Impact**: metrics with high counts have more room for improvement

### Present to the user

Before starting the loop, show:
- The skills that were discovered and what was extracted from each
- The composed fitness profile (metrics, targets, priorities)
- For each metric, whether it will be **scripted**, **semi-scripted**, or **manual**
- Ask: "Does this look right? Anything to add, remove, or reprioritize?"

This is the user's first checkpoint. The second is after instrumentation
(Phase 6), when the scripts are validated.

---

## Phase 6: Instrument

**This phase is non-negotiable and must complete before ANY code changes.**

Every metric that CAN be measured by a script MUST have one. The agent's
subjective assessment of "I counted 5 issues" is not acceptable — a script
that outputs `5` is.

### Why this exists

Without executable scripts, metrics drift:
- The agent counts grep results differently between iterations
- "Fixed" can mean "I think I fixed it" instead of "the script now says 0"
- Re-audits after changes miss regressions the agent doesn't think to check
- The user can't independently verify claims

### Why it must happen before code changes

The scripts capture the "before" snapshot. If you create scripts AFTER
making changes, you have no baseline — you'd need to go back to the
original code to measure it, which is fragile and error-prone. Scripts
first, changes second. Always.

### What to generate

Create `.adaptive-autoresearch/metrics/` at the project root with one script
per metric. Each script:

1. Takes no arguments (all config is inline)
2. Outputs a **single number** to stdout (the metric value)
3. Exits 0 on success, non-zero on measurement failure
4. Is deterministic — same code = same number, every time
5. Runs in under 30 seconds

```bash
# Example: .adaptive-autoresearch/metrics/metric-typescript-any-count.sh
#!/bin/bash
# Counts explicit 'any' types in production code (excludes tests, generated)
grep -rn --include="*.ts" --include="*.tsx" \
  ": any\b\|as any\b" \
  src/ lib/ convex/ \
  --exclude-dir=_generated \
  --exclude-dir=node_modules \
  --exclude-dir=__tests__ \
  | grep -v "\.test\.\|\.spec\." \
  | wc -l
```

```bash
# Example: .adaptive-autoresearch/metrics/metric-auth-coverage.sh
#!/bin/bash
# Counts public query/mutation/action exports without auth checks
python3 -c "
import re, sys, glob

no_auth = 0
for f in glob.glob('convex/**/*.ts', recursive=True):
    if '/_generated/' in f or '/__tests__/' in f:
        continue
    content = open(f).read()
    for m in re.finditer(r'export const (\w+)\s*=\s*(query|mutation|action)\(', content):
        block = content[m.start():m.start()+2000]
        auth_fns = ['requireAuth','getAuthUserId','requirePermission',
                     'requireBotOwnership','requireBotPermission',
                     'requireOrgMembership','requireExecutionAccess',
                     'requireScriptAccess','requireWorkflowOwnership']
        if not any(fn in block for fn in auth_fns):
            no_auth += 1
print(no_auth)
"
```

```bash
# Example: .adaptive-autoresearch/metrics/metric-collect-without-projection.sh
#!/bin/bash
# Counts .collect() calls in query files without a subsequent .map()
python3 -c "
import re, glob
count = 0
for f in glob.glob('convex/domain/**/queries.ts', recursive=True):
    content = open(f).read()
    for m in re.finditer(r'export const (\w+)\s*=\s*query\(', content):
        block = content[m.start():m.start()+3000]
        if '.collect()' in block and '.map(' not in block and 'results.push' not in block:
            count += 1
print(count)
"
```

### Also generate a runner script

Create `.adaptive-autoresearch/run-all.sh` that executes every metric
script from the `metrics/` subdirectory and outputs a JSON summary:

```bash
#!/bin/bash
# Run all metric scripts and output JSON
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
cd "$(dirname "$SCRIPT_DIR")" || exit 1

echo "{"
first=true
for script in "$SCRIPT_DIR"/metrics/metric-*.sh; do
  name=$(basename "$script" .sh | sed 's/^metric-//')
  value=$(bash "$script" 2>/dev/null)
  exit_code=$?
  if [ $exit_code -ne 0 ] || [ -z "$value" ]; then
    value="null"
  fi
  if [ "$first" = true ]; then first=false; else echo ","; fi
  printf '  "%s": %s' "$name" "$value"
done
echo ""
echo "}"
```

### Directory structure

```
.adaptive-autoresearch/
├── fitness.yaml          # fitness profile (metrics, targets, sources)
├── session.md            # living session document
├── session.jsonl         # append-only iteration log
├── run-all.sh            # orchestrator — runs all metrics, outputs JSON
└── metrics/
    ├── metric-<name>.sh  # automated metrics (outputs a number)
    └── check-<name>.sh   # boolean checks (exit 0 = pass, exit 1 = fail)
```

### Classify metrics before scripting

For each metric in the fitness profile, classify it:

| Classification | Action | Example |
|---|---|---|
| **Scriptable** | Generate a script | `any` count, tsc errors, auth coverage |
| **Semi-scriptable** | Script with known false positives, document them | N+1 patterns, query projections |
| **Manual** | Mark as `manual: true` in profile, don't pretend it's automated | "Architecture feels right" |

**Every metric in the profile must have one of these classifications.**
If a metric can't be scripted AND can't be manually verified with clear
criteria, remove it — it's not a real metric.

### Validate the scripts

After generating all scripts, run them and verify:

1. Each script exits 0 and outputs a number
2. The numbers are plausible (not 0 when you know there are issues)
3. Running the same script twice gives the same result
4. Manually spot-check 2-3 metrics against reality

If a script gives wrong results, **fix it before proceeding**. A broken
script is worse than no script — it creates false confidence.

### Present instrumentation to user

Show the user:
- List of metrics and their scripts
- Which metrics are manual (and why)
- Initial run results (pre-baseline, just to validate scripts work)
- "Do these numbers look right?"

This is the second user checkpoint. The first was the fitness profile
(Phase 5). This one validates the measurement apparatus.

---

## Phase 7: Baseline

Run the audit scripts against the current state of the code. This is
iteration zero.

1. Run constraint checks first (build, tests). Fix if broken.
2. **Execute `.adaptive-autoresearch/run-all.sh`** to collect all metrics.
3. Verify script results match reality — spot-check at least 2 metrics
   by manually confirming the number is correct.
4. Record the raw numbers. This is **iteration zero** — the canonical
   "before" snapshot. Every future comparison uses these numbers.
5. Compute total issues (sum of all metrics where lower = better).
6. Write baseline to `.adaptive-autoresearch/session.jsonl` (include full script output).
7. Write initial `.adaptive-autoresearch/session.md` with full context.
8. Show the user where the gaps are.

**This baseline is sacred.** It was captured before any code changes,
using the validated scripts. If scripts are modified later (to fix false
positives, add exclusions, etc.), the baseline MUST be re-captured by
re-running the corrected scripts against the same code state. If the code
has already changed, the old baseline is invalid — note this in the journal.

---

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

---

## Phase 9: The Plan-Driven Loop

The loop executes the plan from Phase 8 instead of picking the weakest
metric each iteration. It does not stop until all groups are complete,
the iteration cap is reached, or the user interrupts.

### Execution model

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
7. NEXT WAVE     — Identify newly unblocked groups, go to SPAWN
```

If the project has only 1-2 issue groups, skip subagent spawning and run
sequentially in main context — parallelism is opportunistic, not mandatory.

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

### The MEASURE step is sacred

**Always run the scripts.** Never eyeball a metric and declare it improved.
The scripts are the single source of truth. If a script says the metric
didn't improve, the metric didn't improve — even if the agent "knows"
it made the right change. Fix the code, not the script.

If during the loop a metric script is found to be wrong (false positives,
missing cases), the agent MAY fix the script — but this counts as its own
iteration with its own journal entry, and the baseline must be recalculated.

### The FOCUS step is key

This is what makes the meta-skill approach work. The agent doesn't try to
hold all knowledge in context. When working on a security group, it
re-reads the security skill. When working on a design system group, it
re-reads the shadcn and web-design-guidelines skills. Just-in-time
knowledge loading, scoped to the current group's domain.

### Keep/discard logic

A change is KEPT only if:
- All constraints pass (tests, build)
- At least one metric improved (number went toward its target)
- No other metric got worse

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

### What the agent should NOT do

- Install skills into the project
- Modify the fitness profile without user approval
- Delete features to improve metrics
- Game metrics (assert-true tests, hidden elements, ts-ignore)
- Continue if stuck for max_no_improvement iterations
- Spawn subagents for groups with file overlap (run sequentially instead)

---

## Session Persistence

Everything lives inside `.adaptive-autoresearch/`:

```
.adaptive-autoresearch/
├── fitness.yaml          # fitness profile (metrics, targets, sources)
├── plan.yaml             # execution plan (groups, ordering, parallelism)
├── session.md            # living session document
├── session.jsonl         # append-only iteration log
├── run-all.sh            # orchestrator
└── metrics/              # one script per metric
```

### `session.jsonl`

Append-only log. One JSON line per iteration with all metric values, the
change description, status (kept/discarded), and reasoning.

### `session.md`

Living document. Includes:
- Project detection results
- Skills that were discovered and what was extracted
- Current metric values (metric | baseline | current | target | script)
- Total issues baseline vs current
- Key wins and dead ends
- Next priorities
- Plan summary and group progress

### `plan.yaml`

Execution plan generated in Phase 8. Contains:
- Issue groups with file:line locations and status tracking
- Dependency graph between groups
- Parallel execution waves
- Progress counters per group

Updated during re-evaluation checkpoints. A fresh agent reads this to
understand what work remains and which groups can run next.

### `metrics/`

Executable metric scripts. These persist across sessions and are the
**canonical measurement apparatus**. A fresh agent:

1. Reads session.md and session.jsonl for context
2. Runs `run-all.sh` to verify current state matches journal
3. If numbers diverge from journal, re-baselines before continuing
4. Does NOT regenerate scripts unless the user asks or a script is broken

The entire `.adaptive-autoresearch/` directory can be committed to git,
run in CI, and reused independently of the agent.

---

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

---

## Communicating with the user

### At the start

Show:
- Detected stack
- Skills discovered (names, install counts, what was extracted)
- Composed fitness profile
- Current gaps
- "Does this look right?"

### After planning (Phase 8)

Show:
- Issue groups with counts and ordering
- Dependency graph (which groups block which)
- Parallel waves (what runs simultaneously)
- Estimated total iterations
- "Does this plan look right? Anything to reorder, merge, or split?"

### During the loop

- Group completion notifications ("auth-coverage: 12/12 fixed, merging")
- Parallel wave progress ("Wave 1: 2/3 groups done")
- Merge success/failure notifications
- Status table every 5 iterations
- Immediate notification on significant wins
- Notification when a metric crosses its target

### On interruption

- Finish current iteration cleanly
- Full status table
- Summary of wins and dead ends
- "Continue, adjust, or stop?"

### On completion

- Before/after comparison
- All kept changes summarized
- Skills that contributed most
- Suggested next steps

---

## References

- `references/discovery-patterns.md` — How to search skills.sh effectively,
  common skill repositories by domain, URL patterns for raw content
