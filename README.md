# adaptive-autoresearch

An autonomous code improvement skill that discovers what "excellent" means for any project, measures it with deterministic scripts, and iterates until it gets there.

Zero built-in opinions. It finds relevant expertise from the [skills.sh](https://skills.sh) ecosystem at runtime, extracts quality criteria, generates executable audit scripts, and runs a keep/discard loop inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch).

Works with any stack, any language, any agent.

## How it works

```
┌──────────────────────────────────────────────────────────────┐
│  0. PREREQS    — Git clean, build passes, skills CLI check   │
│  1. DETECT     — What is this project? Stack, type, deps     │
│  2. DISCOVER   — Search skills.sh for relevant expertise     │
│  3. FETCH      — Read the SKILL.md content from GitHub       │
│  4. EXTRACT    — Pull quality criteria, rules, patterns out  │
│  5. COMPOSE    — Build the fitness profile from all sources  │
│  6. INSTRUMENT — Generate audit scripts, validate them       │
│  7. BASELINE   — Run scripts on UNCHANGED code = iteration 0 │
│  8. LOOP       — Change → measure → keep/discard → repeat    │
└──────────────────────────────────────────────────────────────┘
```

Steps 0–7 happen once, before any code changes. Step 8 runs until all targets are met.

## What makes it different

**Adaptive.** The skill has no hardcoded rules. It discovers what matters for YOUR project by searching community skills relevant to your stack. A Next.js app gets React best practices and accessibility checks. A Python API gets security patterns and test coverage rules. Same skill, different expertise.

**Deterministic measurement.** Every metric has an executable script (`.adaptive-autoresearch/metrics/metric-*.sh`) that outputs a single number. The agent never eyeballs a result — the scripts are the single source of truth. Run them yourself:

```bash
bash .adaptive-autoresearch/run-all.sh
```

**Honest scoring.** No invented composite scores. Raw issue counts, before and after:

```
Metric                              before   after     Δ
typescript-any-count                    38       0   -38 ✅
public-functions-without-auth           16       0   -16 ✅
tsc-errors                              55       0   -55 ✅
n-plus-one-query-loops                   4       0    -4 ✅
TOTAL ISSUES                           155      26   -83%
```

## Installation

```bash
# Install globally (works with any agent)
npx skills add rayoplateado/adaptive-autoresearch -g

# Or project-level
npx skills add rayoplateado/adaptive-autoresearch
```

## Usage

Tell your agent:

- "Run autodev on this project"
- "Optimize this codebase"
- "Make this production-ready"
- "I vibecoded this, clean it up"
- "Audit security and performance"

The skill handles the rest: detects your stack, finds relevant expertise, creates measurement scripts, shows you the plan, and iterates.

## What it produces

After a session, your project has:

| File | Purpose |
|---|---|
| `.adaptive-autoresearch/metrics/` | Deterministic scripts measuring each quality metric |
| `.adaptive-autoresearch/run-all.sh` | Runs all metrics, outputs JSON |
| `.adaptive-autoresearch/fitness.yaml` | The fitness profile (metrics, targets, sources) |
| `.adaptive-autoresearch/session.md` | Living session document with current state |
| `.adaptive-autoresearch/session.jsonl` | Append-only log of every iteration |

The audit scripts persist — commit them, run them in CI, reuse them independently of the agent.

## Key design decisions

**Scripts before changes.** Phase 6 (Instrument) MUST complete before any code is modified. The scripts capture the "before" snapshot. Without this, you have no baseline to compare against.

**Keep/discard, not plan/execute.** Each iteration makes a small change, measures it, and keeps it only if metrics improve without regressions. Bad changes are reverted automatically.

**Just-in-time knowledge.** The agent doesn't hold all skill content in context. When working on security, it loads the security skill. When working on performance, it loads the performance skill. Focused context = better results.

**No composite scores.** The fitness of a codebase is the set of metric values, not a magic number. Total issue count and reduction percentage are the headline stats.

## Agent compatibility

This is a universal skill. It works with any coding agent that can:
- Read files
- Execute bash commands
- Edit code

Tested with: [pi](https://github.com/mariozechner/pi-coding-agent), Claude Code, Cursor, Cline.

## Credits

- Loop design inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch)
- Skill discovery powered by [skills.sh](https://skills.sh)

## License

MIT
