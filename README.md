# relentless

Claude Code sessions die at the context limit and you lose everything. Relentless fixes that.

It wraps the [Claude Agent SDK](https://github.com/anthropics/claude-code-sdk-python) to run headless Claude Code sessions that automatically continue across context limits, detect when they're stuck, and track costs — while retaining access to all Claude Code tools, skills, and MCP servers.

## Why use relentless

**Sessions that don't stop.** Give it a task, walk away, come back to finished work. When context fills up, it checkpoints progress and starts a fresh session that picks up where the last one left off. No manual "here's what I did so far."

**Loop detection.** Without relentless, a stuck agent burns your API budget retrying the same failing approach forever. Relentless catches this — both within a session (same tool call 3x) and across sessions (identical state 2x) — and escalates instead of wasting money.

**Cost tracking.** See exactly what each session cost. Set budget caps. No surprise bills from a runaway agent.

**Crash recovery.** Everything is on disk — `_relentless_state.md`, `_relentless_handoff.json`. If the process dies, `relentless -c` resumes from the last checkpoint. Nothing lives only in the conversation.

**The TDD pipeline.** PRD → gate tests → implement → validate → remediate. Each phase runs in fresh context so there's no drift. The orchestrator chains them and the circuit breaker stops if it's not making progress. You go from "I have an idea" to "tests pass, app works" without babysitting.

### When to use it

- Task takes more than ~10 minutes of agent work
- You want to fire-and-forget (background execution)
- You're implementing from a PRD with multiple acceptance criteria
- You've been burned by context limits mid-task before

### When it's overkill

- Quick questions, small edits, one-file changes
- Anything you'd finish in one Claude Code session anyway

## Install

### As a Claude Code plugin

```bash
claude plugin add dgokeeffe/relentless
```

This gives you all the slash commands (`/relentless-prd`, `/relentless-implement`, `/relentless-orchestrate`, etc.) inside Claude Code.

### CLI only

```bash
curl -fsSL https://raw.githubusercontent.com/dgokeeffe/relentless/main/bin/relentless -o ~/.local/bin/relentless
chmod +x ~/.local/bin/relentless
```

Also grab the system prompt template:

```bash
curl -fsSL https://raw.githubusercontent.com/dgokeeffe/relentless/main/bin/system_prompt.md -o ~/.local/bin/system_prompt.md
```

Requires Python 3.11+ and [uv](https://github.com/astral-sh/uv). Dependencies (`claude-agent-sdk>=0.1.37`) are resolved automatically via uv's inline script metadata.

## Quick start

```bash
# Simple task
relentless "Refactor the auth module to use JWT tokens"

# From a PRD file
relentless -f prd_jwt-auth.md

# More sessions, specific model, verbose
relentless -n 8 -m opus -b 3.0 -v "Build the feature described in PRD.md"

# Continue from where you left off
relentless -c "Keep going"

# Pipe in a task
echo "Fix all lint errors" | relentless
```

## CLI reference

```
relentless [OPTIONS] "TASK"

  -n NUM           Max sessions (default: 5)
  -b USD           Max budget per session in USD
  -m MODEL         Model to use (default: system default)
  -d DIR           Working directory (default: current)
  -f FILE          Read task from file
  -p MODE          Permission mode: default, acceptEdits, plan, auto,
                   bypassPermissions (default: auto)
  -s FILE          State file path (default: ./_relentless_state.md)
  -t MINS          Timeout per session in minutes (default: 30)
  -v               Verbose — show all tool calls and messages
  -c               Continue from existing state file
  --dry-run        Print the prompt without running
  --max-turns N    Max assistant turns per session (default: 50)
  --context-chars N  Estimated char budget per session (default: 480000)
  --failure-repeat-limit N  Same failing tool-result N times = loop
  --adaptive / --no-adaptive  Enable/disable adaptive tuning
  --adaptive-profile MODE  conservative | balanced | aggressive
  --self-test      Run built-in self-tests and exit
```

### Exit codes

| Code | Meaning |
|------|---------|
| 0 | Task completed successfully |
| 1 | Max sessions exhausted without completion |
| 2 | Escalation — stuck, looping, or unresolvable blocker |

## How it works

```
Session 1                    Session 2                    Session N
┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
│ Claude Code      │        │ Claude Code      │        │ Claude Code      │
│ (Agent SDK)      │        │ (Agent SDK)      │        │ (Agent SDK)      │
│                  │        │                  │        │                  │
│ ┌──────────────┐ │        │ ┌──────────────┐ │        │ ┌──────────────┐ │
│ │ Loop         │ │        │ │ Loop         │ │        │ │ Loop         │ │
│ │ Detector     │ │        │ │ Detector     │ │        │ │ Detector     │ │
│ └──────────────┘ │        │ └──────────────┘ │        │ └──────────────┘ │
│                  │        │                  │        │                  │
│ ► all tools,     │        │ ► all tools,     │        │ ► all tools,     │
│   skills, MCPs   │        │   skills, MCPs   │        │   skills, MCPs   │
└────────┬─────────┘        └────────┬─────────┘        └────────┬─────────┘
         │                           │                           │
         ▼                           ▼                           ▼
   _relentless_state.md ──────► _relentless_state.md ──────► _relentless_state.md
   _relentless_handoff.json     _relentless_handoff.json     _relentless_handoff.json
```

Each session runs Claude Code via the Agent SDK with full tool access. Between sessions, relentless:

1. **Checks for completion** — did the agent set `STATUS: COMPLETE` and `success: true`?
2. **Detects loops** — is the state file identical to the last session? (MD5 hash, threshold: 2)
3. **Tracks costs** — accumulates `total_cost_usd` across sessions
4. **Hands off context** — writes a compact handoff JSON so the next session picks up exactly where the last one stopped

### State file format

The agent maintains a markdown state file throughout each session:

```markdown
STATUS: COMPLETE

## Completed
[STEP] Created calculator.py — Calculator class with add/subtract/multiply/divide
[STEP] Created test_calculator.py — 5 pytest tests
[STEP] Ran tests via uv run --with pytest — 5 passed

## In Progress
Working on: complete

## Discoveries
- Environment is uv-managed; must use uv run --with pytest instead of pip install

## Remaining

## Blockers
```

The handoff JSON is a compact structured summary:

```json
{
  "schema_version": "1.0",
  "session": 1,
  "goal": "Create Calculator module and pytest tests, run them",
  "success": true,
  "is_continue": false,
  "summary": "Created calculator.py, test_calculator.py, ran tests — 5 passed.",
  "done": ["Created calculator.py", "Created test_calculator.py", "Ran tests"],
  "in_flight": "",
  "next_step": "",
  "discoveries": ["uv-managed env; use uv run --with pytest"],
  "remaining_acs": [],
  "updated_at": "2026-03-21T22:18:19Z"
}
```

### Loop detection

Two layers prevent wasted compute:

- **Intra-session** (`ToolLoopDetector`): Hashes `(tool_name, input_hash)` pairs. Same tool call 3 times in a session → escalate. Writes to state/handoff files are exempt (they're expected to repeat).
- **Cross-session** (`StateTracker`): MD5-hashes the state file after each session. 2 consecutive identical states → agent is stuck, exit code 2.

### Adaptive tuning

Relentless analyzes run history to adjust parameters automatically:

| Profile | Sessions | Timeout | Tool repeat limit |
|---------|----------|---------|-------------------|
| Conservative | Fewer, shorter | Tight | Low |
| Balanced | Default | Default | Default |
| Aggressive | More, longer | Relaxed | Higher |

Enable with `--adaptive` (on by default). Override with `--adaptive-profile`.

### Context handoff

When a session approaches its context budget (75% warning, 90% hard limit), relentless:

1. Signals the agent to wrap up
2. Runs an optional checkpoint micro-session to finalize the state/handoff files
3. Starts a fresh session with the task + handoff as the prompt

The next session reads the handoff JSON first, then the state file, and resumes from `next_step`.

## TDD pipeline

When installed as a Claude Code plugin, relentless includes a full TDD pipeline:

```
/relentless-preflight  → validate environment, write _relentless_config.json
         ↓
/relentless-prd        → interactive PRD with machine-verifiable acceptance criteria
         ↓
/relentless-gates      → generate failing TDD test stubs from PRD
         ↓
/relentless-implement  → auto-continuing implementation until gates pass
         ↓
/relentless-validate   → live execution QA (API + browser via Chrome DevTools)
         ↓
/relentless-remediate  → targeted fixes for auto-fixable errors
         ↓
/relentless-orchestrate → chains all phases with circuit breaker
```

### Skills

| Skill | What it does |
|-------|-------------|
| `/relentless-preflight` | Validates environment (deps, ports, profiles, PRD structure). Writes `_relentless_config.json` — the contract between all phases. Catches config issues in 30 seconds before the pipeline burns budget. |
| `/relentless-prd` | Interactive PRD generator. Asks clarifying questions, researches the codebase, outputs markdown with Given/When/Then acceptance criteria and a machine-readable Agent Handoff JSON block. |
| `/relentless-gates` | Reads the PRD's Agent Handoff, generates one failing test stub per acceptance criterion (pytest for backend, vitest for frontend). Enriches stubs with real imports and assertion shapes from existing code. Outputs `_gates.json` manifest. |
| `/relentless-implement` | Launches the relentless CLI in the background to implement a PRD or task. Defaults: 5 sessions, verbose, bypass permissions. Reports results when done. |
| `/relentless-validate` | Pure QA — never writes code. Starts the app, hits API endpoints, opens browser routes via Chrome DevTools MCP, captures console errors and network failures. Categorizes each error (config/runtime/permission/auth) and writes `_validation_report.json`. |
| `/relentless-remediate` | Reads the validation report. Patches auto-fixable errors only (config, runtime). One error = one fix. Never validates its own work. |
| `/relentless-orchestrate` | Master coordinator. Runs implement → validate → remediate → re-validate in a loop with fresh context per phase. Circuit breaker stops after 3 loops if error count isn't decreasing. |
| `/relentless-devloop` | Browser testing via Chrome DevTools MCP. Navigates routes, takes screenshots, captures console/network errors. Returns structured JSON for the validator. |
| `/autoresearch` | Autonomous skill optimization. Runs any skill repeatedly, scores outputs against binary evals, mutates the prompt, keeps improvements. Based on Karpathy's autoresearch methodology. |

### Pipeline principles

- **Fresh context per phase.** Each phase runs in an isolated subagent. No accumulated drift from prior phases.
- **No agent judges its own output.** Implementer doesn't validate. Validator doesn't fix. Remediator doesn't verify. This prevents self-reinforcing errors.
- **Artifacts on disk are the only persistence.** `_relentless_state.md`, `_gates.json`, `_validation_report.json`, `_relentless_config.json`. If it's not on disk, it doesn't survive.
- **Circuit breaker.** The orchestrator stops after 3 remediation loops if error count isn't decreasing. Prevents infinite fix-break cycles.
- **Fail fast, fail cheap.** Preflight catches environment issues in 30 seconds. A $0 check saves $10+ of wasted pipeline execution.

## Autoresearch engine

The `bin/autoresearch-engine` script optimizes the system prompt template through automated experimentation:

```bash
# Run baseline only
autoresearch-engine --baseline-only --timeout 8

# Full optimization loop (5 experiments)
autoresearch-engine --max-experiments 5

# Dry run — show config without executing
autoresearch-engine --dry-run
```

It runs relentless on canonical test tasks, scores the output artifacts against 5 binary evals, mutates the system prompt template, and keeps mutations that improve the score. Results are logged to `autoresearch-results/`.

### Evals

| Eval | What it checks |
|------|---------------|
| `state_file_initialized` | State file exists with a valid `STATUS:` line |
| `handoff_json_valid` | Handoff JSON parses with all 15 required keys |
| `task_completed` | `STATUS: COMPLETE` in state + `success: true` in handoff |
| `step_logging` | At least one `[STEP]` entry in state file |
| `discoveries_populated` | Non-empty discoveries for tasks that require them |

### Test tasks

| Task | What it exercises |
|------|------------------|
| `refactor-module` | Create Python module + tests + run + report (4 phases, 10+ tool calls) |
| `analyze-and-report` | Read + analyze + write report with cross-task dependency |
| `iterative-fix` | Create buggy code → test → discover failure → fix → verify |

## Requirements

- Python 3.11+
- [uv](https://github.com/astral-sh/uv)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- Anthropic API key (set `ANTHROPIC_API_KEY` or configure via `claude`)

## License

MIT
