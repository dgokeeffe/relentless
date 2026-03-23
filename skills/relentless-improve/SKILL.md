---
name: relentless-improve
description: "Autonomous improvement engine. Three modes: app (strategic quality improvement via hypothesis-driven code changes), prd (requirement optimization by implementing and evaluating PRDs), self (session log analysis to discover behavioral patterns and write improvement rules). Use when: improve my app, optimize this PRD, self-improve, analyze my sessions, make this better, quality improvement, relentless improve, strategic improvement, autonomous improvement."
user-invocable: true
---

# Relentless Improve

Autonomous improvement engine that combines relentless's fresh-context auto-continuation with autoresearch's hypothesis-driven mutation loop. Three modes, one core loop: **Observe → Evaluate → Hypothesize → Improve → Verify → Repeat.**

Each iteration runs as a fresh relentless CLI session. The progress file (`_progress.md`) is the sole thread of continuity between experiments — plain markdown with "Work Done" and "Work to Do Next" sections. No context drift. No accumulated bias.

---

## mode detection

Parse the user's invocation:
- `/relentless-improve app` or `/relentless-improve app /path/to/project` → App mode
- `/relentless-improve prd` or `/relentless-improve prd prd_slug.md` → PRD mode
- `/relentless-improve self` → Self-improvement mode
- Bare `/relentless-improve` → ask: "Which mode? (app / prd / self)"

---

## crash recovery (all modes)

Every invocation starts the same way:

```bash
cat _improve_results/_progress.md 2>/dev/null || echo "NO_STATE"
```

**If progress file exists and Status line says `running`:** Read the "Work Done" section to see completed experiments and the "Work to Do Next" section for the pending hypothesis. Resume from there. Tell the user: "Resuming from experiment {N}. Best score so far: {best_pass_rate}%."

**If progress file exists and Status line says `complete` or `plateau`:** Ask: "Previous run finished at {best_pass_rate}%. Start fresh or review results?"

**If no progress file:** Proceed to mode-specific context gathering.

This is the unified recovery pattern — same for all modes, no special codepaths.

---

## progress file format

The `_progress.md` file is the sole thread of continuity. Plain markdown, not JSON.

```markdown
# Progress: relentless-improve ({mode})
## Status: running | complete | plateau
## Target: {path or description}
## Best: {score}/{max} ({pass_rate}%) at experiment {N}

## Config
- Mode: {app|prd|self}
- Evals: {list}
- Budget: {N or unlimited}

## Work Done
- exp 1: [keep] Added error boundaries — score 3/5 (60%)
- exp 2: [discard] Tried pagination — no improvement
- exp 3: [keep] Fixed responsive layout — score 4/5 (80%)

## Work to Do Next
- Next hypothesis: API error responses need structured error codes
- Evals still failing: API health, lint errors
- Plateau counter: 1/3
```

**Why markdown, not JSON:**
No schema parsing, no nested object traversal. The "Work Done" list is the experiment history. The "Work to Do Next" section is the handoff instruction. That is the entire contract.

---

## mode 1: app — strategic quality improvement

### what this is

NOT bug fixing — that's what `relentless-remediate` does. This is **strategic improvement**: rethinking architecture, improving UX patterns, adding defensive coding, optimizing performance. Each experiment implements a hypothesis-driven change and measures whether it improved overall quality.

### context gathering

1. Find project root: `git rev-parse --show-toplevel`
2. Detect stack: check for `backend/`, `frontend/`, `pyproject.toml`, `package.json`
3. Find test commands: `uv run pytest` (Python), `npm test` (JS), or ask user
4. Find lint commands: `ruff check` (Python), `npx eslint .` (JS), or ask user
5. Ask user: "What's the focus? (general / ui-polish / performance / accessibility / data-integration / autonomous / or describe your own)"
6. Ask user: "Budget cap? (max experiments, default: no cap)"
7. If autonomous: ask user "Check-in frequency? (every N experiments / none — default: none)"
   - `none` = fully autonomous, runs until nothing left to improve or budget hit
   - A number = pauses every N experiments to show progress and ask to continue

Select eval preset based on focus area. User can always customize.

### eval presets

#### preset: general (default)

For functional correctness, code quality, and basic stability.

```
EVAL 1: Zero test failures
Question: Do all existing tests pass with zero failures?
Pass: All tests pass (exit code 0)
Fail: Any test failure or error

EVAL 2: Zero console errors
Question: Does the app run with zero JavaScript console errors?
Pass: list_console_messages returns no errors
Fail: Any console error present

EVAL 3: Null safety
Question: Do data-dependent components handle null/undefined without crashing?
Pass: No TypeError or "Cannot read properties of undefined" in console
Fail: Any null-related error

EVAL 4: API health
Question: Do all API endpoints respond with 2xx status within 2 seconds?
Pass: All endpoints return 2xx in <2s
Fail: Any 4xx/5xx or timeout

EVAL 5: Zero lint errors
Question: Does the codebase pass linting with zero errors?
Pass: Linter exits with code 0
Fail: Any lint error (warnings are OK)
```

#### preset: ui-polish

For visual refinement — overflow, layout, responsive design, data display, and design consistency. Uses Chrome DevTools MCP heavily.

```
EVAL 1: No horizontal overflow
Question: Does any page have a horizontal scrollbar at the default viewport width?
Pass: evaluate_script returns false for: document.documentElement.scrollWidth > document.documentElement.clientWidth
Fail: Any page has horizontal overflow
How to check: For each route, navigate_page → evaluate_script("document.documentElement.scrollWidth > document.documentElement.clientWidth")

EVAL 2: No clipped or truncated content
Question: Are there elements where text or content is visually cut off by overflow:hidden?
Pass: No elements have scrollHeight significantly larger than clientHeight (>20px difference) among visible text containers
Fail: Any visible text container has clipped content
How to check: evaluate_script to find all elements matching "p, span, td, li, h1, h2, h3, h4, div" where el.scrollHeight > el.clientHeight + 20 and getComputedStyle(el).overflow !== 'visible'

EVAL 3: Responsive layout integrity
Question: Does the layout work without overlap or breakage at both 1280px and 768px widths?
Pass: At both viewport widths — no horizontal overflow, no overlapping elements, all navigation accessible
Fail: Layout breaks at either width
How to check: resize_page to 1280x800 → take_screenshot → check overflow eval. Then resize_page to 768x1024 → take_screenshot → check overflow eval. Compare screenshots for obvious breakage.

EVAL 4: Data tables and lists render correctly
Question: Do all data-driven components (tables, lists, cards) show real data from the API — not empty states, loading spinners stuck, or error messages?
Pass: Every data component shows populated content. No "No data", "Loading...", "Error", or empty table bodies visible after page load completes.
Fail: Any data component shows empty/error/stuck-loading state
How to check: navigate_page → wait_for("table, [class*=list], [class*=card]") → take_screenshot → evaluate_script to check that tables have >0 rows, lists have >0 items

EVAL 5: No overlapping elements
Question: Are there any elements that visually overlap each other (z-index conflicts, absolute positioning errors, margin collapse)?
Pass: No unintentional element overlap detected
Fail: Any elements overlap that shouldn't
How to check: evaluate_script to get bounding rects of major layout elements (nav, main, sidebar, footer, modals) and check for intersection. Also take_screenshot and check visually.

EVAL 6: Console and network clean
Question: Is the app free of console errors AND failed network requests?
Pass: Zero console errors AND zero network requests with status >= 400
Fail: Any console error or failed network request
How to check: list_console_messages (filter level=error) AND list_network_requests (filter status >= 400)
```

**UI-polish hypothesis generation strategy:**

When analyzing failing evals, map to CSS/layout fix categories:

| Failing Eval | Hypothesis Category | Example Fix |
|---|---|---|
| Horizontal overflow | Container width / flex/grid layout | "Main content area uses fixed width — switch to max-width with overflow-x handling" |
| Clipped content | overflow:hidden on wrong container | "Card components clip long descriptions — add text-overflow: ellipsis or expand height" |
| Responsive breakage | Missing media queries / rigid layouts | "Sidebar doesn't collapse on mobile — add responsive breakpoint at 768px" |
| Empty data components | Loading state / API integration | "Table renders before API response arrives — add loading skeleton and data-ready check" |
| Overlapping elements | z-index / positioning / margin | "Modal backdrop doesn't cover the sidebar — fix z-index stacking context" |
| Console/network errors | Runtime bugs affecting rendering | "Failed API call causes component to render error state instead of data" |

**Each relentless session for ui-polish MUST:**
1. Use Chrome DevTools MCP tools directly: `navigate_page`, `take_screenshot`, `evaluate_script`, `resize_page`, `list_console_messages`, `list_network_requests`
2. Take before/after screenshots for every change
3. Test at BOTH 1280px and 768px viewports
4. Save all screenshots to `_improve_results/exp_{N:03d}/screenshots/`

#### preset: performance

For response times, rendering speed, bundle size, and resource efficiency.

```
EVAL 1: API response time
Question: Do all API endpoints respond in under 1 second?
Pass: Every endpoint returns within 1000ms
Fail: Any endpoint takes >1000ms
How to check: list_network_requests → filter API calls → check timing field

EVAL 2: Page load time
Question: Does the initial page load complete (DOMContentLoaded) in under 3 seconds?
Pass: DOMContentLoaded fires within 3000ms
Fail: Takes longer than 3000ms
How to check: evaluate_script("performance.timing.domContentLoadedEventEnd - performance.timing.navigationStart")

EVAL 3: No large bundle warnings
Question: Is the JavaScript bundle under 500KB (gzipped)?
Pass: No chunk exceeds 500KB gzipped
Fail: Any chunk over threshold
How to check: list_network_requests → filter JS files → check transfer size

EVAL 4: No unnecessary re-renders
Question: Are there zero React/framework "too many re-renders" warnings in the console?
Pass: No re-render warnings
Fail: Any re-render or performance warning in console
How to check: list_console_messages → filter for "re-render", "Maximum update depth", "performance"

EVAL 5: Smooth interactions
Question: Do click/form interactions respond within 200ms (no visible lag)?
Pass: UI updates within 200ms of user action
Fail: Any interaction feels sluggish or takes >200ms to update
How to check: click element → evaluate_script timestamp before/after → check delta
```

#### preset: accessibility

For WCAG compliance, keyboard navigation, screen reader support, and color contrast.

```
EVAL 1: No accessibility violations
Question: Does the page pass a Lighthouse accessibility audit with score >= 90?
Pass: Lighthouse accessibility score >= 90
Fail: Score < 90
How to check: lighthouse_audit with categories=["accessibility"]

EVAL 2: All images have alt text
Question: Do all <img> elements have non-empty alt attributes?
Pass: Every img has alt text
Fail: Any img missing alt
How to check: evaluate_script("document.querySelectorAll('img:not([alt]), img[alt=\"\"]').length === 0")

EVAL 3: Keyboard navigation works
Question: Can all interactive elements (buttons, links, inputs) be reached via Tab key?
Pass: Tab order visits all interactive elements without getting stuck
Fail: Any interactive element unreachable via keyboard
How to check: evaluate_script to count focusable elements, then press_key Tab repeatedly and track focus movement

EVAL 4: Color contrast sufficient
Question: Do all text elements meet WCAG AA contrast ratio (4.5:1 for normal text, 3:1 for large)?
Pass: All text passes contrast requirements
Fail: Any text below minimum contrast
How to check: lighthouse_audit → extract contrast violations from results

EVAL 5: Form labels present
Question: Do all form inputs have associated <label> elements or aria-label attributes?
Pass: Every input has a label
Fail: Any input without label association
How to check: evaluate_script("document.querySelectorAll('input:not([aria-label]):not([id]), select:not([aria-label]):not([id])').length") and check for matching labels
```

#### preset: data-integration

For backend-frontend data flow — API contracts, error states, loading states, and data consistency.

```
EVAL 1: All API endpoints return valid JSON
Question: Do all API endpoints return properly structured JSON responses?
Pass: Every endpoint returns parseable JSON with expected schema
Fail: Any endpoint returns malformed JSON, HTML error pages, or unexpected schema
How to check: list_network_requests → filter API calls → evaluate response content-type and body

EVAL 2: Error states handled gracefully
Question: When an API returns an error (4xx/5xx), does the UI show a user-friendly message instead of crashing?
Pass: Error responses produce clear UI feedback (toast, error boundary, message)
Fail: App crashes, shows raw error, or silently fails
How to check: evaluate_script to intercept fetch and return error → take_screenshot → check for error UI

EVAL 3: Loading states present
Question: Do data-dependent components show a loading indicator before data arrives?
Pass: Every async data component shows a spinner/skeleton before content
Fail: Any component shows empty/blank state during load
How to check: navigate_page → take_screenshot immediately (before data loads) → check for loading indicators

EVAL 4: Data consistency
Question: Does the data displayed in the frontend match what the API returns?
Pass: Counts, values, and labels in the UI match API response payloads
Fail: Any mismatch between API data and rendered content
How to check: list_network_requests → capture API response → evaluate_script to read DOM values → compare

EVAL 5: Pagination/filtering works
Question: Do filters, search, and pagination update the displayed data correctly?
Pass: Applying filter/search/page change updates the view with correct data
Fail: Stale data, wrong results, or no update
How to check: fill filter input → click → list_network_requests for new API call → take_screenshot → verify data changed
```

#### preset: custom

When the user describes a specific focus area (e.g., "improve the quality of architecture diagrams"), build custom evals dynamically.

**Process for custom preset:**

1. Ask the user: "Describe what 'good' looks like for your specific use case."
2. From their description, extract 4-6 binary yes/no quality checks
3. For each check, determine the Chrome DevTools verification method:
   - Visual output quality → `take_screenshot` + `evaluate_script` to inspect rendered elements
   - Generated content → `evaluate_script` to read DOM content
   - File output → check file exists and validate structure
   - API output → `list_network_requests` to capture response
4. Present the proposed evals to the user for confirmation before starting
5. Proceed with the standard experiment loop

**Example: "improve architecture diagrams generated by nano banana pro"**

User says: "My app uses nano banana pro to draw architecture diagrams. The diagrams sometimes have overlapping labels, the connections between nodes are hard to follow, and the layout doesn't use space efficiently."

Generated custom evals:
```
EVAL 1: No overlapping labels
Question: Are all node labels fully visible and non-overlapping in the rendered diagram?
Pass: No text elements overlap other text or node boundaries
Fail: Any label overlaps another label or is partially hidden behind a node
How to check: take_screenshot of rendered diagram → evaluate_script to get bounding rects of all text/label elements → check for intersection

EVAL 2: Connection paths are traceable
Question: Can every connection between nodes be visually followed from source to target without ambiguity?
Pass: All edges/arrows have clear paths, arrowheads visible, no crossing ambiguity
Fail: Edges overlap in confusing ways or arrowheads are hidden
How to check: take_screenshot → evaluate_script to count edge elements and verify each has distinct path data

EVAL 3: Space utilization
Question: Does the diagram use the available canvas area efficiently (no large empty regions while nodes are cramped)?
Pass: Nodes are distributed across the canvas with consistent spacing
Fail: Nodes bunched in one corner or area while >40% of canvas is empty
How to check: evaluate_script to get all node positions → calculate bounding box vs canvas size → check distribution

EVAL 4: Diagram renders without errors
Question: Does the diagram render completely with zero console errors?
Pass: Canvas/SVG renders fully, no errors in console
Fail: Rendering errors, missing elements, or console exceptions
How to check: navigate_page → wait_for(diagram container) → list_console_messages → take_screenshot

EVAL 5: Text is legible
Question: Are all labels, annotations, and titles readable at the default zoom level?
Pass: All text elements have font size >= 12px and sufficient contrast
Fail: Any text is too small to read or has poor contrast against its background
How to check: evaluate_script to get computed font-size and color/background of all text elements
```

#### preset: autonomous

The agent decides what to improve. Most powerful mode, but needs layered safety.

**How it works: two phases.**

**Phase 1: Discovery (read-only — ZERO code changes)**

A dedicated relentless session explores the app without modifying anything. It:

1. Reads the codebase structure, key files, README, package.json/pyproject.toml
2. Runs all existing tests — records pass/fail counts
3. Runs the app and navigates every route via Chrome DevTools
4. Takes screenshots of every page at 1280px and 768px
5. Checks console errors, network failures, accessibility (lighthouse_audit)
6. Reads code quality: looks for anti-patterns, TODO comments, error handling gaps, dead code
7. Checks performance: page load times, API response times, bundle sizes

From all of this, it produces a **discovery report** — a prioritized list of improvement opportunities:

```json
{
  "discovery_timestamp": "ISO-8601",
  "app_summary": "Brief description of what the app does",
  "existing_test_count": 42,
  "existing_test_pass_rate": 95.2,
  "routes_found": ["/", "/dashboard", "/settings"],
  "screenshots": ["discovery/route_home_1280.png", ...],
  "opportunities": [
    {
      "id": "OPP-1",
      "priority": "high",
      "category": "ui",
      "title": "Dashboard table overflows container on mobile",
      "description": "The demand plan table has no responsive handling — content clips at <900px",
      "evidence": "screenshot discovery/route_dashboard_768.png shows horizontal overflow",
      "proposed_eval": "No horizontal overflow on any route at 768px width",
      "estimated_effort": "small",
      "risk": "low"
    },
    {
      "id": "OPP-2",
      "priority": "high",
      "category": "error-handling",
      "title": "API errors show raw stack traces to user",
      "description": "When /api/estimates returns 500, the frontend renders the raw error object instead of a friendly message",
      "evidence": "Simulated error via evaluate_script — screenshot shows JSON blob",
      "proposed_eval": "All API error responses produce user-friendly UI, never raw JSON/stack traces",
      "estimated_effort": "medium",
      "risk": "low"
    },
    {
      "id": "OPP-3",
      "priority": "medium",
      "category": "performance",
      "title": "Estimates list loads 500+ records without pagination",
      "description": "GET /api/estimates returns all records. Slow on large datasets and causes browser jank",
      "proposed_eval": "API supports pagination and frontend uses it, initial load <100 records",
      "estimated_effort": "medium",
      "risk": "medium"
    }
  ]
}
```

The discovery session prompt:

```
You are running the DISCOVERY phase of relentless-improve (autonomous mode).
Project root: {project_root}

YOUR TASK: Explore this app thoroughly and identify improvement opportunities.
You must NOT modify any code. Read-only exploration only.

Steps:
1. Read project structure: key files, README, config, package.json/pyproject.toml
2. Run existing tests: {test_commands} — record results
3. Start the app and explore via Chrome DevTools:
   - navigate_page to every route you can find
   - take_screenshot at 1280x800 and 768x1024 for each route
   - list_console_messages — record all errors
   - list_network_requests — record all failures
   - lighthouse_audit for accessibility and performance scores
4. Read source code looking for:
   - Error handling gaps (unhandled promise rejections, missing try/catch)
   - UI issues (hardcoded widths, missing responsive breakpoints, no loading states)
   - Performance issues (N+1 queries, missing pagination, large bundles)
   - Code quality (dead code, TODO comments, duplicated logic)
   - Security issues (XSS vectors, missing input validation, exposed secrets)
5. Prioritize by: impact (high/medium/low) × effort (small/medium/large)
6. Write discovery report to _improve_results/discovery.json

IMPORTANT: Do NOT modify any files. This is a read-only exploration.
Write ONLY to _improve_results/discovery.json and _improve_results/discovery/ (screenshots).
```

**Phase 2: Continuous improvement loop**

After discovery, the agent works through opportunities automatically, then re-discovers to find more. This is the full autonomous cycle:

```
CYCLE:
  1. DISCOVER — explore app, find opportunities, prioritize
  2. IMPROVE  — work through opportunities (high priority, low risk first)
  3. RE-DISCOVER — explore again post-improvements, find NEW opportunities
  4. REPEAT until:
     - Re-discovery finds zero new opportunities (app is clean)
     - Budget cap hit
     - Plateau: two consecutive discovery cycles find the same issues
```

If `check_in_interval` is set (not `none`), pause every N experiments and show progress. Otherwise run fully autonomously — the guardrails are the safety net, not human approval.

**On first discovery:** If check-in is `none`, start improving immediately. If check-in is set, present the discovery list once and ask which to pursue, then run uninterrupted until the next check-in.

**Re-discovery triggers** after all current opportunities are resolved (kept or skipped after failed attempts). The re-discovery session is identical to the initial discovery but produces a diff:
- NEW opportunities (not seen before)
- RESOLVED opportunities (previously found, now fixed)
- PERSISTENT opportunities (still present despite attempts — skip these)

**Stopping conditions for the continuous loop:**
```
STOP if:
- Re-discovery finds 0 new opportunities (nothing left to improve)
- Two consecutive discovery cycles find identical opportunity sets (plateau)
- Budget cap reached (total experiments across all cycles)
- All remaining opportunities are high-risk and check_in=none
  (high risk ALWAYS requires user confirmation, even in continuous mode)
```

**Mandatory safety guardrails (always active, cannot be disabled):**

```
SAFETY GUARDRAILS (always active in autonomous mode):

GUARDRAIL 1: Regression gate (hard block)
  Before scoring any experiment, run ALL existing tests: {test_commands}
  If any previously-passing test now fails → DISCARD immediately.
  No exceptions. No "the test was flaky." Discard and move on.

GUARDRAIL 2: Screenshot regression check
  Compare after-screenshots with discovery-phase screenshots for ALL routes,
  not just the route being improved. If any UNRELATED route looks different
  (layout shift, missing elements, broken styling) → DISCARD.

GUARDRAIL 3: Console error ceiling
  Count console errors after change. If the count is HIGHER than discovery
  baseline → DISCARD. Improvements must not introduce new errors anywhere.

GUARDRAIL 4: Scope boundary
  Each experiment touches ONE opportunity. Do not "while I'm here" fix
  adjacent things. One hypothesis, one change, one eval.

GUARDRAIL 5: Periodic check-in (optional)
  If check_in_interval is set (not "none"):
    Every {check_in_interval} experiments, PAUSE and report progress:

    "Completed {N} experiments. {kept} kept, {discarded} discarded.
     Opportunities resolved: {resolved_count}/{total_found}.
     Changes made:
     - OPP-1: Fixed (dashboard table now responsive)
     - OPP-2: Fixed (error boundary added)
     - OPP-3: In progress
     Continue? (yes / stop / skip to next opportunity)"

    Wait for user response before continuing.

  If check_in_interval is "none":
    No pausing. Run fully autonomously. Log progress to
    _improve_results/progress.md after each experiment so the user
    can check status anytime by reading the file.

GUARDRAIL 6: Risk-aware ordering
  Work through opportunities in this order:
  1. Low risk, high priority first (safest wins)
  2. Medium risk only after low-risk opportunities exhausted
  3. High risk opportunities ALWAYS require explicit user confirmation,
     even in continuous/no-check-in mode. This is the ONE exception
     to fully autonomous operation — high risk changes are never auto-applied.
     If no user is available, skip the high-risk opportunity and continue.
```

**Autonomous mode evals are dynamic** — each opportunity has its own eval (from `proposed_eval` in the discovery report), PLUS the universal regression guardrails always apply.

Effective eval suite per experiment in autonomous mode:
```
EVAL 1: (from opportunity) — does the specific improvement work?
EVAL 2: Regression gate — all existing tests still pass?
EVAL 3: Console ceiling — no new console errors introduced?
EVAL 4: Visual regression — no unrelated routes changed?
```

This means the max score per experiment = 4. An experiment must score 4/4 to be kept. Any regression = instant discard, even if the improvement itself worked.

**Re-discovery session prompt (used after all current opportunities are resolved):**

```
You are running RE-DISCOVERY for relentless-improve (autonomous mode).
Project root: {project_root}
Discovery cycle: {cycle_number}

This is NOT the first discovery. Previous discoveries found and resolved:
{resolved_opportunities_summary}

Opportunities that persisted despite attempts (SKIP these):
{persistent_opportunities}

YOUR TASK: Explore the app again with fresh eyes. The codebase has changed
since last discovery. Look for:
1. NEW issues introduced by recent improvements
2. Issues that are NOW visible because other issues were fixed
3. Deeper quality issues that weren't obvious before
4. Integration issues between the improvements that were made

Use the same exploration process as initial discovery.
Compare against the discovery baseline screenshots in _improve_results/discovery/.

Write to _improve_results/rediscovery_{cycle_number}.json with:
- new_opportunities: issues not seen in any prior discovery
- resolved: issues from prior discovery that are now fixed
- persistent: issues that still exist despite prior attempts

IMPORTANT: Do NOT re-report opportunities that were already found and
either fixed or marked as persistent. Only report genuinely NEW findings.
```

**Continuous loop state tracking:**

The progress file tracks discovery cycles as flat markdown under an "Autonomous Discovery" heading:

```markdown
## Autonomous Discovery
- Check-in: none
- Current cycle: 2
- Total opportunities: 12 found, 8 resolved, 2 persistent, 2 remaining

### Cycle 1
- Opportunities: 8 found, 6 resolved, 1 persistent, 1 skipped
- Experiments: 8 run, 6 kept

### Cycle 2
- Opportunities: 4 found, 2 resolved, 1 persistent, 1 remaining
- Experiments: 3 run, 2 kept
```

User can replace or extend eval presets. Max 6 evals per preset (autonomous mode adds its guardrail evals on top).

### what each relentless session does

**Baseline (experiment 0):** Score the app as-is. No changes — just run tests, devloop, linter. Record baseline.

**Experiment N (N > 0):** The session receives:

```
You are running experiment {N} of relentless-improve (app mode).
Project root: {project_root}

## Context
- Baseline score: {baseline_pass_rate}%
- Current best: {best_pass_rate}% (experiment {best_experiment})
- Last experiment: {last_status} — {last_description}
- Improvements that worked: {kept_mutations_summary}
- Changes that didn't help: {discarded_mutations_summary}

## Your hypothesis
{hypothesis}

## Your task
1. Implement the strategic improvement described in the hypothesis
2. Make ONE focused change — do not fix unrelated things
3. Run tests: {test_commands}
4. Run linter: {lint_commands}
5. Start the app and run /relentless-devloop to validate frontend
6. Write all results to _improve_results/exp_{N:03d}/:
   - test_results.txt (test output)
   - lint_results.txt (lint output)
   - devloop_results.json (browser test results)
   - summary.json (your assessment: what changed, what improved, what regressed)

## Important
- Strategic improvement, not bug patching. Think about WHY things break, not just WHAT broke.
- Do not refactor broadly. One hypothesis, one change.
- If the hypothesis doesn't apply (e.g., the app already handles this case), write that in summary.json and exit.
```

**For ui-polish preset**, the session prompt adds:

```
## UI-Polish specific instructions
Focus: {focus_area}
Eval preset: ui-polish

You MUST use Chrome DevTools MCP tools directly for evaluation:
1. navigate_page to each route in the app
2. take_screenshot BEFORE making any changes (save to exp_{N:03d}/screenshots/before/)
3. evaluate_script to check for overflow, clipped content, overlapping elements
4. resize_page to 768x1024 and take_screenshot to check responsive layout
5. list_console_messages and list_network_requests for error checking
6. After implementing your change, take_screenshot AFTER (save to exp_{N:03d}/screenshots/after/)
7. Re-run all evaluate_script checks to verify improvement

Your eval artifact output MUST include:
- screenshots/before/*.png — every route at 1280px and 768px BEFORE changes
- screenshots/after/*.png — every route at 1280px and 768px AFTER changes
- overflow_check.json — result of overflow evaluate_script per route
- responsive_check.json — result of responsive checks at both viewport sizes
- console_network.json — console errors and failed network requests
```

### git worktree strategy (keep/discard)

Before each experiment:
```bash
git worktree add .improve-exp-{N:03d} -b improve/exp-{N:03d}
```

The relentless session runs inside the worktree directory (`-d` flag).

After scoring:
- **Keep:** Merge back to working branch, clean up worktree
  ```bash
  git merge improve/exp-{N:03d}
  git worktree remove .improve-exp-{N:03d}
  git branch -D improve/exp-{N:03d}
  ```
- **Discard:** Delete worktree and branch
  ```bash
  git worktree remove .improve-exp-{N:03d} --force
  git branch -D improve/exp-{N:03d}
  ```

---

## mode 2: prd — requirement optimization

### what this is

The PRD is the "prompt" for implementation. Just like autoresearch improves a SKILL.md by running it and scoring the output, this mode improves a PRD by implementing from it and scoring the implementation.

### context gathering

1. Find PRD: argument or `ls prd_*.md` (if one, use it; if multiple, ask)
2. Read the PRD — understand its ACs, scope, Agent Handoff block
3. Ask user: "What does a successful implementation look like beyond passing gate tests?"
4. Ask user: "Budget cap? (max experiments, default: 10)"

### default evals

```
EVAL 1: ACs are machine-verifiable
Question: Can every acceptance criterion be tested by a script, not a human?
Pass: All ACs tagged (Machine-verifiable) in the PRD
Fail: Any AC requires manual verification without a skip marker

EVAL 2: First-attempt gate pass rate
Question: What percentage of gate tests pass on the first implementation attempt?
Pass: >= 80% of gates pass without remediation
Fail: < 80% pass rate

EVAL 3: Scope is bounded
Question: Does the PRD have an explicit Non-Goals section with at least 3 items?
Pass: Non-Goals section present with 3+ items
Fail: Missing or fewer than 3

EVAL 4: No ambiguous ACs
Question: Is every AC specific enough that two developers would implement it the same way?
Pass: All ACs use Given/When/Then format with concrete values
Fail: Any AC uses vague language ("fast", "good", "appropriate")

EVAL 5: Agent Handoff complete
Question: Does the Agent Handoff JSON include validation_surfaces, constraints, and escalate_on?
Pass: All three fields present and non-empty
Fail: Any field missing or empty
```

### what each relentless session does

**Two phases per experiment:**

**Phase A — Implementation (relentless session):**
```
You are testing PRD quality for experiment {N} of relentless-improve (prd mode).

Implement the feature described in this PRD from scratch in a clean worktree.
PRD file: {prd_file}
Work directory: {worktree_path}

Steps:
1. Run /relentless-gates to generate gate tests from the PRD
2. Implement until all gates pass
3. Run /relentless-devloop if frontend surfaces exist
4. Write results to _improve_results/exp_{N:03d}/:
   - gate_results.txt (pytest output)
   - implementation_log.md (what you did, what was hard, what was ambiguous)
   - devloop_results.json (if applicable)
```

**Phase B — PRD Mutation (relentless session):**
```
You are the PRD analyst for experiment {N} of relentless-improve (prd mode).

Implementation results from Phase A:
- Gate pass rate: {X}/{Y}
- Implementation difficulty: {summary from implementation_log}
- Ambiguous areas: {list from implementation_log}

Prior PRD mutations that helped: {kept_list}
Prior PRD mutations that didn't help: {discarded_list}

Hypothesis: {hypothesis about what the PRD got wrong}

Task: Edit the PRD at {prd_file} to fix ONE identified weakness.
- Sharpen ambiguous ACs with concrete values
- Add missing edge cases
- Improve Agent Handoff with better constraints
- Do NOT rewrite the entire PRD

Write changes to _improve_results/exp_{N:03d}/prd_diff.md
```

### git worktree strategy

Same as app mode. Each implementation attempt runs in a worktree. PRD mutations are committed to the main branch (the PRD is the artifact being improved, not the implementation).

---

## mode 3: self — behavioral self-improvement

### what this is

Analyze past Claude Code session logs to find patterns: repeated corrections, wasted tool calls, context waste, repeated mistakes. Synthesize actionable rules. Persist to CLAUDE.md and MCP memory.

### context gathering

1. List available projects:
   ```bash
   ls -d ~/.claude/projects/*/ | while read d; do
     count=$(find "$d" -name "*.jsonl" -maxdepth 2 | wc -l)
     echo "$(basename "$d"): $count sessions"
   done
   ```
2. Ask user: "Which project(s) to analyze?" (present the list)
3. Read current CLAUDE.md rules and MCP memory entries (to avoid duplicates)
4. Ask user: "Focus area? (corrections / tool waste / false starts / all)" Default: all
5. Ask user: "Budget cap? (max experiments, default: 5)"

### default evals

```
EVAL 1: Pattern is concrete
Question: Does the analysis identify a specific, actionable pattern (not a vague observation)?
Pass: Pattern names specific tools, files, or behaviors
Fail: Pattern is generic ("could be more efficient")

EVAL 2: Evidence-backed
Question: Is each pattern supported by 2+ specific examples from session logs?
Pass: Each pattern has timestamps and quoted content from logs
Fail: Pattern is unsupported or has only 1 example

EVAL 3: Rule is novel
Question: Is the proposed rule NOT already in CLAUDE.md or MCP memory?
Pass: No existing rule covers this pattern
Fail: Duplicate or near-duplicate of existing rule

EVAL 4: Rule is actionable
Question: Does the rule start with NEVER or ALWAYS and include a specific trigger?
Pass: Rule follows format "NEVER/ALWAYS <action> when <condition>. Why: <reason>"
Fail: Rule is vague or missing trigger condition
```

### what each relentless session does

**Session logs are JSONL format.** Key record types:
- `type: "user"` → user messages (look for corrections: "no", "don't", "stop", "wrong")
- `type: "progress"` → tool calls and results (look for wasted reads, repeated tool calls)
- Fields: `uuid`, `timestamp`, `sessionId`, `parentUuid`, `message.content`

**Each experiment analyzes a batch of 3-5 session logs** (logs can be enormous — fresh context per batch):

**Phase A — Analysis session:**
```
You are analyzing Claude Code session logs for self-improvement patterns.
This is experiment {N} of relentless-improve (self mode).

Session logs to analyze (read these JSONL files):
{list of 3-5 absolute paths to .jsonl files}

Look for these pattern types:
1. **Corrections** — user says "no", "don't", "stop", "wrong", "not that" after an assistant action
2. **Tool waste** — Read/Glob/Grep calls whose results are never referenced again
3. **False starts** — tool call immediately followed by a completely different approach
4. **Repeated mistakes** — same wrong assumption across multiple sessions
5. **Context bloat** — reading entire files when only a section was needed

Existing rules to NOT re-discover:
{current CLAUDE.md learned rules section}
{recent MCP memory entries}

Patterns already found in prior experiments:
{prior_patterns_summary}

Output to _improve_results/exp_{N:03d}/analysis.json:
{{
  "sessions_analyzed": ["session-id-1", "session-id-2"],
  "patterns": [
    {{
      "type": "correction|waste|false_start|mistake|bloat",
      "description": "specific description",
      "evidence": [
        {{"session": "uuid", "timestamp": "ISO", "detail": "quoted content"}}
      ],
      "frequency": 3,
      "proposed_rule": "NEVER/ALWAYS ... Why: ...",
      "target": "claude_md|memory|skill:<name>"
    }}
  ]
}}
```

**Phase B — Application session:**
```
You are applying self-improvement rules from experiment {N}.

Patterns to apply (only those that passed eval scoring):
{filtered_patterns}

Tasks:
1. For target=claude_md: append rule to CLAUDE.md "Learned Rules" section
2. For target=memory: call MCP save_memory with key="learning/rule/<slug>"
3. For target=skill:<name>: read the skill SKILL.md and make a targeted improvement

Record what was changed in _improve_results/exp_{N:03d}/application.json:
{{
  "rules_written": [{{"target": "...", "rule": "...", "location": "..."}}],
  "rules_skipped": [{{"pattern": "...", "reason": "..."}}]
}}
```

---

## core loop (all modes)

Once context is gathered and evals are defined, this loop runs identically for all modes.

### setup

1. Create `_improve_results/` directory
2. Initialize `_improve_results/_progress.md` with mode, target, evals, empty Work Done section
3. Generate `_improve_results/dashboard.html` (see dashboard section)
4. Initialize `_improve_results/results.json` with empty experiment list
5. Open dashboard: `open _improve_results/dashboard.html`

### baseline (experiment 0)

Run the mode-specific baseline:
- **App:** Run tests + linter + devloop. No code changes.
- **PRD:** Score the PRD structure against evals. No implementation yet.
- **Self:** Read existing CLAUDE.md rules and memory. Score current coverage.

Score against all evals. Record in _progress.md and results.json. Update dashboard.

**If baseline >= 95%:** Ask user "Baseline is already at {score}%. Continue optimization?"

### experiment loop

```
NEVER STOP. Run autonomously until a stopping condition is met.

for experiment_num in 1, 2, 3, ...:

    1. READ PROGRESS: Read _improve_results/_progress.md

    2. ANALYZE: Look at which evals are failing most.
       Read actual failing artifacts from the last experiment.
       Identify the pattern — what's causing the most failures?

    3. HYPOTHESIZE: Form ONE hypothesis about what to improve.
       - Good: "API error responses lack structured error codes, causing
               frontend components to show generic 'Something went wrong' messages"
       - Bad: "Make the app better"
       The hypothesis must be specific and testable.

    4. PREPARE WORKTREE (app/prd modes):
       git worktree add .improve-exp-{N:03d} -b improve/exp-{N:03d}

    5. LAUNCH RELENTLESS SESSION:
       env -u CLAUDECODE -u CLAUDE_CODE_ENTRYPOINT relentless \
         -d {worktree_or_project_path} \
         -n 3 -t 20 -p bypassPermissions -v \
         "{mode-specific task prompt with hypothesis}" \
         > _improve_results/exp_{N:03d}/relentless.log 2>&1

    6. WAIT: Monitor for completion. Read log tail periodically.

    7. SCORE: Read experiment artifacts. Apply each eval. Calculate total score.
       - For each eval: read the relevant artifact, answer the yes/no question
       - Total score = count of passes
       - Pass rate = (total_score / num_evals) * 100

    8. DECIDE:
       if score > best_score:
           STATUS = "keep"
           best_score = score
           best_experiment = experiment_num
           plateau_counter = 0
           → Merge worktree (app/prd modes)
       else:
           STATUS = "discard"
           plateau_counter += 1
           → Delete worktree (app/prd modes)

    9. UPDATE PROGRESS: Rewrite _improve_results/_progress.md

    10. UPDATE DASHBOARD: Write to _improve_results/results.json

    11. LOG: Append to _improve_results/changelog.md:
        ## Experiment {N} — {keep/discard}
        **Score:** {score}/{max} ({pass_rate}%)
        **Hypothesis:** {hypothesis}
        **Change:** {what was actually done}
        **Result:** {which evals improved/declined}
        **Remaining failures:** {what still fails}

    12. CHECK CIRCUIT BREAKERS:
        - plateau_counter >= 3 → STOP (plateau)
        - experiment_num >= budget_cap → STOP (budget)
        - pass_rate == 100% for 2 consecutive → STOP (perfect)
        - pass_rate >= 95% and plateau_counter >= 2 → STOP (diminishing returns)
```

---

## eval scoring

All modes use binary evals. The orchestrator scores — NOT the relentless session (no agent judges its own output).

**Scoring process:**
1. Read the experiment's output artifacts
2. For each eval, read the relevant artifact and answer the yes/no question
3. Pass = 1, Fail = 0
4. Total score = sum of passes across all evals
5. Max score = number of evals (since runs_per_experiment = 1 for all modes)
6. Pass rate = (total_score / max_score) * 100

**The orchestrator must be explicit about scoring.** For each eval, state:
- Which artifact was checked
- What was found
- Pass or fail
- Why

---

## dashboard

Generate `_improve_results/dashboard.html` — a single self-contained HTML file with inline CSS and JavaScript.

**Dashboard features:**
- Auto-refreshes every 10 seconds from `results.json`
- Mode badge: App / PRD / Self (colored)
- Score progression line chart (Chart.js from CDN)
- Colored experiment bars: green = keep, red = discard, blue = baseline
- Per-eval breakdown table
- Current status: "Running experiment N..." or "Complete" or "Plateau detected"
- Cumulative cost tracker (from relentless session costs)
- Experiment count and keep rate

**`results.json` format:**
```json
{
  "mode": "app",
  "target": "project-name",
  "status": "running",
  "current_experiment": 3,
  "baseline_score": 60.0,
  "best_score": 80.0,
  "total_cost_usd": 2.45,
  "experiments": [
    {
      "id": 0,
      "score": 3,
      "max_score": 5,
      "pass_rate": 60.0,
      "status": "baseline",
      "hypothesis": null,
      "description": "original state — no changes",
      "cost_usd": 0.50,
      "duration_sec": 120
    }
  ],
  "eval_breakdown": [
    {"name": "Zero test failures", "pass_count": 3, "total": 4}
  ]
}
```

Open the dashboard immediately after creating it: `open _improve_results/dashboard.html`

Update `results.json` after every experiment so the dashboard stays current.

When the run finishes, update `status` to `"complete"` or `"plateau"`.

---

## circuit breaker

```
After each experiment:

  if score > best_score:
      plateau_counter = 0
      best_score = score
  else:
      plateau_counter += 1

  STOP if:
  - plateau_counter >= 3 (score not improving)
  - experiment_num >= budget_cap (user-defined limit)
  - pass_rate == 100% for 2 consecutive experiments (perfect)
  - pass_rate >= 95% and plateau_counter >= 2 (diminishing returns)
```

On stop, update `_progress.md` Status line to `complete` or `plateau` and proceed to results delivery.

---

## deliver results

When the loop stops (any stopping condition), present:

1. **Score trajectory:** Baseline {baseline}% → Best {best}% ({improvement}% improvement)
2. **Experiments:** {total} run, {kept} kept, {discarded} discarded ({keep_rate}% keep rate)
3. **Top improvements:** The 3 mutations that helped most (from changelog)
4. **Remaining weaknesses:** What the evals still catch (from last experiment)
5. **Cost:** Total ${cost} across all relentless sessions
6. **Artifacts location:**
   ```
   _improve_results/
   ├── _progress.md           # Handoff state (Work Done / Work to Do Next)
   ├── results.json           # Dashboard data
   ├── dashboard.html         # Open to review
   ├── changelog.md           # Every experiment logged
   └── exp_*/                 # Per-experiment artifacts
   ```

For **self mode**, also present:
- Rules written to CLAUDE.md: {count}
- Rules saved to MCP memory: {count}
- Sessions analyzed: {count} out of {total available}

---

## hypothesis generation strategy

The hardest part. Here is the strategy per mode:

### app mode

Analyze which evals fail most. Map to improvement categories:

| Failing Eval | Hypothesis Category | Example |
|---|---|---|
| Test failures | Code correctness | "The data transformation pipeline doesn't handle edge case X" |
| Console errors | Null safety, missing imports | "Components assume API always returns arrays, but it returns null on empty" |
| API health | Performance, error handling | "The /estimates endpoint runs an N+1 query pattern" |
| Lint errors | Code quality | "Unused imports accumulated from refactoring" |

**Hypothesis must be strategic, not tactical:**
- BAD: "Fix the TypeError on line 47"
- GOOD: "Data-dependent components lack a consistent loading/error/empty pattern, causing cascading null reference errors. Introduce a shared DataRenderer wrapper."

### prd mode

Analyze what went wrong in implementation:
- Gates that failed → AC was ambiguous or underspecified
- Remediation needed → Missing error cases in PRD
- Too many sessions → Scope too broad, needs decomposition

### self mode

Statistical analysis of session logs:
- Count corrections per pattern type
- Count tool calls per session that were never used downstream
- Identify "false starts" (approach abandoned within 3 tool calls)
- Find most common correction phrases

---

## the test

A good relentless-improve run:

1. **Started with a baseline** — measured before changing anything
2. **Used binary evals** — no scales, no vibes
3. **Made strategic improvements** — not just bug fixes
4. **Changed one thing per experiment** — so you know what helped
5. **Kept a complete log** — every experiment in changelog.md
6. **Improved the score** — measurable improvement from baseline
7. **Ran autonomously** — didn't stop to ask between experiments
8. **Stopped when appropriate** — plateau detection, not arbitrary loop count

---

## how this connects to other skills

**What feeds into relentless-improve:**
- Existing app code (app mode)
- PRDs from `/relentless-prd` (prd mode)
- Session history (self mode — auto-accumulated)

**What relentless-improve feeds into:**
- Improved app code → ready for deployment
- Improved PRDs → feed into `/relentless-implement` for better implementations
- Improved rules → better performance in all future sessions
- Changelog → any future agent can continue the optimization

**Skills used by relentless-improve:**
- `relentless` CLI — auto-continuation engine for each experiment
- `/relentless-devloop` — browser testing (app mode)
- `/relentless-validate` — live execution validation (app mode)
- `/relentless-gates` — gate generation (prd mode)
- MCP memory server — rule persistence (self mode)
