# Safety Layer Design

## Summary

Add a cross-cutting safety layer to adaptive-autoresearch that validates
fetched skills and generated metric scripts before they can influence code
changes or execute on the user's machine.

## Motivation

The skill fetches community-published SKILL.md content from skills.sh,
extracts criteria from it, and generates executable shell scripts — all
without validating that the content is safe. A malicious or poorly-written
skill could cause harmful script generation, data exfiltration, or
destructive code changes.

## Design Decisions

- **Layered defense**: Gate at fetch time (trust validation) + gate at
  script generation (pattern review)
- **Warning-based trust**: No hard allowlists. Install count + age as
  signals. Below thresholds → warn user, let them decide.
- **Automatic with escalation**: Clear violations are surfaced, but the
  user always has the final say. No absolute rejections.
- **On by default**: `safety.yaml` generated with sensible defaults on
  first run. Disable explicitly with `enabled: false`.
- **Cross-cutting section**: Safety defined once in SKILL.md, referenced
  from Phase 3 and Phase 6 with one-line callouts. No new phases.

## Configuration: `safety.yaml`

```yaml
# .adaptive-autoresearch/safety.yaml
enabled: true

trust:
  min_installs: 5000
  min_age_days: 30

scripts:
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
  allowed_write_paths:
    - .adaptive-autoresearch/
    - /tmp/

overrides:
  approved_skills: []
  approved_scripts: []
```

- Trust thresholds use AND: both must be below to trigger a warning.
- Blocked patterns are regexes applied line-by-line to generated scripts.
- Write operations to `allowed_write_paths` don't trigger warnings.
- Overrides accumulate as the user approves exceptions.
- Approved scripts tracked by checksum — re-flagged if content changes.

## Gate 1: Skill Trust Validation (Phase 3)

After fetching each skill's metadata, before reading its SKILL.md:

1. Collect: publisher, install count, publish/first-commit date
2. Check against `trust.min_installs` and `trust.min_age_days`
3. Meets thresholds → proceed silently, log `trust: auto`
4. Below thresholds → warn user with name, installs, age
5. User approves → add to `overrides.approved_skills`, proceed
6. User rejects → skip skill, note gap in session.md

User can request "details" to see first 20 lines of the SKILL.md before
deciding. No hard blocks — every skill can be approved by the user.

## Gate 2: Script Safety Review (Phase 6)

After generating each metric script, before first execution:

1. Read script line by line
2. Match against `scripts.blocked_patterns`
3. Exclude matches targeting `allowed_write_paths`
4. No matches → proceed, mark `review: clean`
5. Matches found → show user: script name, flagged lines, matched patterns
6. User approves → add to `overrides.approved_scripts` with checksum
7. User rejects → mark metric as `manual: true`, skip automation
8. User chooses "edit" → re-run safety check on edited version

Runs once per script. After approval, not re-flagged unless checksum changes.

## Integration Points

### Phase 0 (Prerequisites)
Add: "If `safety.yaml` does not exist, generate with defaults."

### Phase 3 (Fetch)
Add after fetching: "Before reading a fetched skill's content, run the
trust validation gate (see Safety Layer). Skip rejected skills."

### Phase 6 (Instrument)
Add after generation: "Before executing any generated script for the first
time, run the script safety review gate (see Safety Layer). Mark rejected
scripts as manual metrics."

## Edge Cases

- **Resuming**: Overrides persist in `safety.yaml`. Only new skills or
  modified scripts (checksum mismatch) trigger review.
- **All skills rejected for a domain**: Agent notes gap in session.md,
  continues with reduced coverage.
- **All scripts rejected for a metric**: Metric becomes `manual: true`.
  Reported with `?` in status tables.
- **Non-interactive mode (CI/background)**: Below-threshold skills and
  flagged scripts are rejected by default. Only auto-approved content runs.
- **Committed overrides**: `safety.yaml` in `.adaptive-autoresearch/` can
  be committed — team shares trust decisions.

## What Doesn't Change

- Phase ordering and numbering
- The keep/discard loop (Phase 9) — scripts are validated before it starts
- Session resumption logic (overrides prevent re-prompting)
- Metric script format or fitness.yaml structure
