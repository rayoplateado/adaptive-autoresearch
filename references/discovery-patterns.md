# Discovery Patterns

How to discover relevant skills using the `skills` CLI, identify the best
candidates, and fetch their content for any project type.

---

## The `skills` CLI

The primary tool for discovery. Already installed and available globally.

### Core discovery commands

```bash
# Interactive search — browse and select
skills find

# Keyword search — returns matching skills with metadata
skills find <keyword>
skills find next.js
skills find security
skills find "api design"

# List skills in a specific repository without installing
skills add owner/repo -l
skills add vercel-labs/agent-skills -l
skills add shadcn/ui -l

# Check what's already installed
skills list              # project-level
skills list -g           # global
skills ls --json         # machine-readable output
```

### Installing skills

```bash
# Install all skills from a repo
skills add owner/repo -g -y

# Install specific skills from a repo
skills add owner/repo -s skill-name -g -y

# Install to all agents
skills add owner/repo --all
```

### Checking for updates

```bash
skills check             # see what's outdated
skills update            # update all to latest
```

---

## Known high-quality repositories by domain

These publishers consistently produce actionable, well-maintained skills:

| Domain | Publisher | Repository | Notes |
|--------|-----------|-----------|-------|
| React / Next.js | vercel-labs | agent-skills | 200K+ installs |
| UI components | shadcn | ui | shadcn/ui patterns |
| Design quality | pbakaus | impeccable | Polish & critique |
| Security | supercent-io | skills-template | Multi-domain |
| Testing | obra | superpowers | TDD, debugging |
| Performance | supercent-io | skills-template | Multi-domain |
| Accessibility | supercent-io | skills-template | Multi-domain |
| Mobile / Expo | expo | skills | Native UI, data |
| Vue | antfu, hyf0 | skills, vue-skills | Vue ecosystem |
| SEO | coreyhaines31 | marketingskills | Content & SEO |
| API design | wshobson | agents | REST, patterns |
| Supabase | supabase | agent-skills | Supabase stack |
| Playwright | currents-dev | playwright-best-practices-skill | E2E testing |
| Python | wshobson | agents | Backend patterns |
| General quality | anthropics | skills | Anthropic official |
| Code architecture | obra | superpowers | Architecture |

To explore any of these:

```bash
skills add vercel-labs/agent-skills -l    # see what's inside
skills add supercent-io/skills-template -l
skills add obra/superpowers -l
```

---

## Mapping project type to search queries

### Frontend (React/Next.js + Tailwind + shadcn)

```bash
skills find react
skills find next.js
skills find tailwind
skills find shadcn
skills find accessibility
skills find "web design"
skills find performance
skills find security
```

Key repos to explore:
- `vercel-labs/agent-skills`
- `shadcn/ui`
- `pbakaus/impeccable`
- `supercent-io/skills-template`

### API / Backend (Node.js)

```bash
skills find nodejs
skills find "api design"
skills find security
skills find testing
skills find performance
skills find database
```

Key repos:
- `wshobson/agents`
- `supercent-io/skills-template`
- `obra/superpowers`

### Full-stack (Next.js + API)

Run both frontend and backend searches. Also:

```bash
skills find authentication
skills find "full stack"
```

### Mobile (React Native / Expo)

```bash
skills find expo
skills find "react native"
skills find "native ui"
```

Key repos:
- `expo/skills`
- `vercel-labs/agent-skills`

### Python backend (FastAPI, Django)

```bash
skills find python
skills find fastapi
skills find django
skills find security
skills find testing
```

### Bot / Scraper

```bash
skills find playwright
skills find scraping
skills find debugging
skills find performance
```

---

## Fetching raw SKILL.md from GitHub (when not installing)

Given a skill at `owner/repo/skill-name`, try these URLs in order:

```
1. https://raw.githubusercontent.com/{owner}/{repo}/main/skills/{skill-name}/SKILL.md
2. https://raw.githubusercontent.com/{owner}/{repo}/main/{skill-name}/SKILL.md
3. https://raw.githubusercontent.com/{owner}/{repo}/main/.claude/skills/{skill-name}/SKILL.md
```

Special cases:
- `anthropics/skills` → `main/skills/{skill-name}/SKILL.md`
- `vercel-labs/agent-skills` → `main/{skill-name}/SKILL.md`

### Fetching referenced files

If a SKILL.md references `references/some-doc.md`, fetch from the same
base path:

```
https://raw.githubusercontent.com/{owner}/{repo}/main/skills/{skill-name}/references/some-doc.md
```

Only fetch referenced files with actionable quality criteria. Skip examples,
templates, and assets.

---

## How many skills to fetch

Aim for **5-10 skills** total, covering domains relevant to the project.

- More than 10 → noise, conflicting advice, context bloat
- Fewer than 3 → missed quality dimensions

Prioritize by install count. A skill with 200K installs has been
community-validated far more than one with 500.

---

## Handling conflicting advice

Different skills may contradict each other. Resolution order:

1. **More installs wins** — community consensus
2. **More specific wins** — a shadcn skill trumps a generic React skill
   on UI component questions
3. **User preference wins** — explicit user direction is final
4. **When in doubt, flag it** — ask the user to resolve

---

## Refreshing knowledge

The agent caches skill content in `autodev-session.md` so it doesn't
re-fetch every session. But the user can say "refresh the skills" to
trigger a new discovery + fetch cycle.

Also useful:

```bash
skills check    # see if installed skills have updates
skills update   # pull latest versions
```

Skills evolve. A periodic refresh (every few weeks or when starting a
major new phase) is healthy.
