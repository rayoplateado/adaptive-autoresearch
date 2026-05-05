# Safety Layer Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a cross-cutting safety layer that validates fetched skills (trust gate) and generated metric scripts (script review gate) before they execute.

**Architecture:** A new "Safety Layer" section in SKILL.md defines the rules and two gate points. Phase 0 generates `safety.yaml` with defaults. Phase 3 references Gate 1 (trust). Phase 6 references Gate 2 (script review). No new phases or renumbering.

**Tech Stack:** Markdown (SKILL.md edits), YAML (safety.yaml schema)

---

### Task 1: Add Safety Layer Section to SKILL.md

**Files:**
- Modify: `SKILL.md:50-68` (insert new section between Phase 0 heading and its content)

**Step 1: Insert the Safety Layer section after line 49 (end of overview diagram) and before Phase 0**

Insert this section between the `---` separator after the overview and `## Phase 0: Prerequisites`:

```markdown
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
```

**Step 2: Verify the section renders correctly**

Read the file and confirm the new section sits between the overview diagram
and Phase 0, with proper markdown formatting.

**Step 3: Commit**

```bash
git add SKILL.md
git commit -m "add Safety Layer section to SKILL.md"
```

---

### Task 2: Add safety.yaml Generation to Phase 0

**Files:**
- Modify: `SKILL.md` — Phase 0: Prerequisites section (around line 56 after Task 1's insertion)

**Step 1: Add step 4 to Phase 0's verification list**

After the existing three prerequisite checks (skills CLI, git clean, build
passes), add:

```markdown
4. **`safety.yaml` exists** — if `.adaptive-autoresearch/safety.yaml` does
   not exist, generate it with the defaults defined in the Safety Layer
   section above. If it exists, load it and respect its configuration
   throughout the session.
```

**Step 2: Commit**

```bash
git add SKILL.md
git commit -m "add safety.yaml generation to Phase 0 prerequisites"
```

---

### Task 3: Add Gate 1 Callout to Phase 3

**Files:**
- Modify: `SKILL.md` — Phase 3: Fetch section

**Step 1: Add trust validation callout**

After the "Strategy: local first, remote second" subsection and before
"Context management", insert:

```markdown
### Trust validation

Before reading a fetched skill's SKILL.md content, run Gate 1 (Skill Trust
Validation) from the Safety Layer section. If the user rejects a skill,
skip it entirely and note the coverage gap in session.md. Do not extract
criteria from rejected skills.
```

**Step 2: Commit**

```bash
git add SKILL.md
git commit -m "add trust validation gate callout to Phase 3"
```

---

### Task 4: Add Gate 2 Callout to Phase 6

**Files:**
- Modify: `SKILL.md` — Phase 6: Instrument section

**Step 1: Add script review callout**

After the "Also generate a runner script" subsection and before "Validate
the scripts", insert:

```markdown
### Script safety review

Before executing any generated script for the first time, run Gate 2
(Script Safety Review) from the Safety Layer section. Scripts the user
rejects become `manual: true` metrics in the fitness profile — they are
still tracked but not automatically measured. Scripts the user approves
are checksummed in `safety.yaml` and not re-reviewed unless modified.
```

**Step 2: Commit**

```bash
git add SKILL.md
git commit -m "add script safety review gate callout to Phase 6"
```

---

### Task 5: Update Directory Structure Documentation

**Files:**
- Modify: `SKILL.md` — the directory structure block in Phase 6 and Session Persistence

**Step 1: Add safety.yaml to the directory listing in Phase 6**

Update the directory structure block to include `safety.yaml`:

```
.adaptive-autoresearch/
├── fitness.yaml          # fitness profile (metrics, targets, sources)
├── safety.yaml           # trust thresholds, script blocklist, overrides
├── session.md            # living session document
├── session.jsonl         # append-only iteration log
├── run-all.sh            # orchestrator — runs all metrics, outputs JSON
└── metrics/
    ├── metric-<name>.sh  # automated metrics (outputs a number)
    └── check-<name>.sh   # boolean checks (exit 0 = pass, exit 1 = fail)
```

**Step 2: Update the Session Persistence section's directory listing**

Same addition — add `safety.yaml` with comment.

**Step 3: Commit**

```bash
git add SKILL.md
git commit -m "add safety.yaml to directory structure documentation"
```

---

### Task 6: Update Resume Logic for Safety Overrides

**Files:**
- Modify: `SKILL.md` — "Resuming a session" section

**Step 1: Add safety.yaml to the resume checklist**

In the numbered resume steps (currently 1-8), add after reading plan.yaml:

```markdown
5. Read `safety.yaml` for trust overrides (skip re-prompting for
   previously approved skills and scripts)
```

Renumber subsequent steps accordingly.

**Step 2: Commit**

```bash
git add SKILL.md
git commit -m "add safety.yaml to session resume logic"
```

---

### Task 7: Update README.md "What it produces" Table

**Files:**
- Modify: `README.md` — the "What it produces" table

**Step 1: Add safety.yaml row to the table**

Add after the `session.jsonl` row:

```markdown
| `.adaptive-autoresearch/safety.yaml` | Trust config, script blocklist, approved overrides |
```

**Step 2: Commit**

```bash
git add README.md
git commit -m "add safety.yaml to README output table"
```

---

### Task 8: Final Review

**Step 1: Read the full SKILL.md and verify**

- Safety Layer section is between overview and Phase 0
- Phase 0 references safety.yaml generation
- Phase 3 has trust validation callout
- Phase 6 has script review callout
- Directory listings include safety.yaml
- Resume logic includes safety.yaml
- No broken markdown, no renumbered phases

**Step 2: Read README.md and verify the table is correct**

**Step 3: Run a consistency check**

```bash
grep -n "safety" SKILL.md | head -20
grep -n "safety" README.md
```

Verify all references are consistent.

**Step 4: Final commit if any fixups needed**

```bash
git add -A
git commit -m "fix: consistency pass for safety layer integration"
```
