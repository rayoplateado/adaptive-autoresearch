# adaptive-autoresearch session: acme-dashboard

## Project

| Field | Value |
|---|---|
| Name | acme-dashboard |
| Type | Frontend SPA |
| Stack | React 19 + Vite + TypeScript + Tailwind v4 + shadcn/ui |
| Backend | Convex |
| Tests | None (added vitest during loop) |

## Skills Discovered

| Skill | Installs | What we extracted |
|---|---|---|
| vercel-labs/.../nextjs-react-best-practices | 228K | TypeScript strictness, useEffect anti-patterns, component size |
| shadcn/ui/shadcn | 28K | Design token usage, color hardcoding, component patterns |
| pbakaus/impeccable/impeccable | 15K | Visual spacing, hierarchy, responsive checks |
| supercent-io/.../security-best-practices | 13K | XSS patterns, input validation, auth |
| currents-dev/.../playwright-best-practices | 8K | Test structure, coverage patterns |
| local/experie-design-system | local | Token compliance, dark mode patterns |

## Metrics — Current State (after iteration 15)

```
Metric                          type          baseline   current   target    Δ        status
─────────────────────────────────────────────────────────────────────────────────────────────
typescript-any-count            direct              23         0        0   -23  ✅  DONE
tsc-errors                      direct              41         0        0   -41  ✅  DONE
hardcoded-color-values          direct              67         4        0   -63  🔧  in progress
console-log-count               direct              18         0        0   -18  ✅  DONE
useeffect-derived-state         direct               5         0        0    -5  ✅  DONE
oversized-components            proxy                8         2        0    -6  🔧  in progress
prop-heavy-components           proxy                6         1        0    -5  🔧  in progress
accessibility-violations        tool-visual         34        11        0   -23  🔧  in progress
lighthouse-performance          tool-visual         52        74       90   +22  🔧  in progress
visual-spacing-consistency      visual              12         3        0    -9  🔧  in progress
visual-hierarchy                visual               4         1        0    -3  🔧  in progress
responsive-breakpoints          visual               7         2        0    -5  🔧  in progress
─────────────────────────────────────────────────────────────────────────────────────────────
TOTAL ISSUES (lower-is-better)                     225        21             -204  (-91%)
```

## Key Wins

### Iteration 1-3: TypeScript cleanup
Replaced all 23 `any` types with proper interfaces. Most were API response
types from Convex — created `types/api.ts` with shared interfaces extracted
from actual query returns. Fixed 41 tsc errors (mostly downstream of the
any→interface changes). Zero regressions.

### Iteration 4: Console cleanup
Removed 18 `console.log` calls. 3 were in error handlers — replaced with
proper error boundary logging. Rest were debug leftovers.

### Iteration 5: useEffect purge
5 `useEffect` calls were computing derived state. Replaced with `useMemo`
(2 cases) and lifted computation into the render path (3 cases — the
derivation was trivial). Build time improved slightly.

### Iteration 6-9: Design tokens
Replaced 63 of 67 hardcoded color values with Tailwind theme tokens.
4 remaining are in a third-party chart library wrapper where the API
requires hex strings — documented as `wont_fix` with inline comment
explaining why.

### Iteration 10-12: Visual spacing (VISUAL METRICS IN ACTION)
This is where it got interesting. The visual-spacing-consistency metric
screenshot+rubric found 12 spacing violations at baseline:

> **Vision analysis (iteration 10, before changes):**
> Screenshot: dashboard main view at 1280x720
> - Sidebar nav items: 14px gap (not on 4px grid → should be 12px or 16px)
> - Card grid: mix of gap-3 (12px) and gap-4 (16px) across pages
> - Section headings: margin-bottom varies (20px, 24px, 28px — inconsistent)
> - Form labels: 6px gap to input (should be 8px)
> Violations counted: 12

After fixing padding/margin/gap values to consistent Tailwind scale values:

> **Vision analysis (iteration 12, after changes):**
> Screenshot: dashboard main view at 1280x720
> - Sidebar nav items: 16px gap ✓
> - Card grid: consistent gap-4 (16px) ✓
> - Section headings: consistent mb-6 (24px) ✓
> - Form labels: gap-2 (8px) ✓
> - Minor: stats card internal padding looks slightly tight (not a grid violation)
> Violations counted: 3

Score went 12 → 3. The remaining 3 are borderline calls — within tolerance.

### Iteration 13-15: Component splitting + accessibility
Split 6 of 8 oversized components. The `DashboardPage.tsx` (847 lines) became:
- `DashboardHeader.tsx` (62 lines)
- `MetricsGrid.tsx` (124 lines)
- `RecentActivity.tsx` (89 lines)
- `DashboardCharts.tsx` (156 lines)
- `DashboardPage.tsx` (43 lines — composition only)

Accessibility violations dropped from 34 to 11 after adding:
- aria-labels to icon buttons (axe: button-name)
- alt text to avatar images (axe: image-alt)
- role attributes to custom dropdowns (axe: listbox pattern)

## Dead Ends

### Iteration 8 (DISCARDED)
Tried to replace all `text-gray-600` with `text-muted-foreground` but
the shadcn token has different opacity behavior in dark mode. Visual
regression detected by responsive-breakpoints metric (dark mode screenshot
showed invisible text). Reverted.

### Iteration 14 (DISCARDED)
Attempted to split `SettingsForm.tsx` (312 lines) but the form state was
deeply intertwined. The split version had 2 new tsc errors from circular
imports. Constraint check failed → auto-reverted. Marked as `stuck` —
needs refactor of form state management first.

## Plan Progress

```
Group                          Status       Progress
──────────────────────────────────────────────────────
types-and-strictness           ✅ completed  23/23 fixed
console-cleanup                ✅ completed  18/18 fixed
useeffect-antipatterns         ✅ completed  5/5 fixed
design-tokens                  🔧 in progress  63/67 fixed (4 wont_fix)
component-architecture         🔧 in progress  6/8 fixed (1 stuck)
accessibility                  🔧 in progress  23/34 fixed
visual-polish                  🔧 in progress  9/12 resolved
lighthouse-performance         ⏳ pending    blocked by accessibility
```

## Next Priorities

1. Finish accessibility violations (11 remaining — mostly color contrast)
2. Lighthouse should jump once contrast is fixed (overlaps with a11y)
3. Remaining 2 oversized components — may need state management refactor
4. Visual hierarchy final pass (1 remaining violation in settings page)
