# relentless

Auto-continuing Claude Code agent with TDD pipeline, loop detection, cost tracking, and adaptive tuning.

`relentless` wraps the [Claude Agent SDK](https://github.com/anthropics/claude-code-sdk-python) to run headless Claude Code sessions that automatically continue across context limits, detect when they're stuck, and track costs — while retaining access to all Claude Code tools, skills, and MCP servers.

## Install

### As a Claude Code plugin

```bash
claude plugin add dgokeeffe/relentless
```

This gives you all the slash commands (`/relentless-prd`, `/relentless-implement`, `/relentless-orchestrate`, etc.) inside Claude Code.

### CLI only

Copy the `bin/relentless` script to your PATH:

```bash
curl -fsSL https://raw.githubusercontent.com/dgokeeffe/relentless/main/bin/relentless -o ~/.local/bin/relentless
chmod +x ~/.local/bin/relentless
```

Requires Python 3.11+ and [uv](https://github.com/astral-sh/uv). Dependencies (`claude-agent-sdk>=0.1.37`) are resolved automatically via uv's inline script metadata.

## Quick start

```bash
# Simple task
relentless "Refactor the auth module to use JWT tokens"

# From a file
relentless -f task.md

# With options
relentless -n 8 -m opus -b 3.0 -v "Build the feature described in PRD.md"

# Continue from where you left off
relentless -c "Keep going"

# Pipe in a task
echo "Fix all lint errors" | relentless
```

## CLI options

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
```

## How it works

```
Session 1                    Session 2                    Session N
┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
│ Claude Code      │        │ Claude Code      │        │ Claude Code      │
│ (Agent SDK)      │        │ (Agent SDK)      │        │ (Agent SDK)      │
│                  │        │                  │        │                  │
│ ┌──────────────┐ │        │ ┌──────────────┐ │        │ ┌──────────────┐ │
│ │ Tool Loop    │ │        │ │ Tool Loop    │ │        │ │ Tool Loop    │ │
│ │ Detector     │ │        │ │ Detector     │ │        │ │ Detector     │ │
│ └──────────────┘ │        │ └──────────────┘ │        │ └──────────────┘ │
│                  │        │                  │        │                  │
│ ► tools, skills, │        │ ► tools, skills, │        │ ► tools, skills, │
│   MCP servers    │        │   MCP servers    │        │   MCP servers    │
└────────┬─────────┘        └────────┬─────────┘        └────────┬─────────┘
         │                           │                           │
         ▼                           ▼                           ▼
   _relentless_state.md ──────► _relentless_state.md ──────► _relentless_state.md
   (progress checkpoint)        (progress checkpoint)        (final state)
```

Each session runs Claude Code via the Agent SDK with full tool access. Between sessions, relentless:

1. **Checks for completion** — did the agent signal it's done?
2. **Detects loops** — is the state file identical to the last session? (MD5 hash comparison, threshold: 2 identical hashes)
3. **Tracks costs** — accumulates `total_cost_usd` across sessions
4. **Hands off context** — writes a continuation prompt with the state file so the next session picks up where the last left off

### Loop detection

Two layers prevent wasted compute:

- **Intra-session** (`ToolLoopDetector`): Hashes `(tool_name, input_hash)` pairs. If the same tool call repeats 3 times in a session, it escalates.
- **Cross-session** (`StateTracker`): MD5-hashes the state file after each session. If 2 consecutive sessions produce identical state, the agent is stuck and relentless stops with exit code 2.

### Adaptive tuning

Relentless analyzes run history to adjust parameters automatically:

| Profile | Sessions | Timeout | Tool repeat limit |
|---------|----------|---------|-------------------|
| Conservative | Fewer, shorter | Tight | Low |
| Balanced | Default | Default | Default |
| Aggressive | More, longer | Relaxed | Higher |

Enable with `--adaptive` (on by default). Override with `--adaptive-profile`.

## TDD Pipeline (plugin skills)

When installed as a Claude Code plugin, relentless includes a full TDD pipeline:

```
/relentless-preflight  → validate environment, write config
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

| Skill | Description |
|-------|-------------|
| `relentless-preflight` | Environment validation. Checks deps, ports, profiles, PRD structure. Writes `_relentless_config.json`. |
| `relentless-prd` | Interactive PRD generator. Outputs markdown with Given/When/Then acceptance criteria and machine-readable Agent Handoff JSON. |
| `relentless-gates` | TDD gate test generator. Reads PRD, outputs failing pytest/vitest stubs + `_gates.json` manifest. |
| `relentless-implement` | Launches relentless CLI to implement a PRD or task. Runs in background with progress tracking. |
| `relentless-validate` | Pure QA agent. Hits APIs, opens browser routes, captures errors. Never writes code. Outputs `_validation_report.json`. |
| `relentless-remediate` | Targeted code fixer. Reads validation report, patches auto-fixable errors only. One error = one fix. |
| `relentless-orchestrate` | Master orchestrator. Chains implement → validate → remediate → re-validate with fresh context per phase and circuit breaker. |
| `relentless-devloop` | Browser testing via Chrome DevTools MCP. Navigation, screenshots, console/network error capture. |
| `autoresearch` | Autonomous skill optimization. Runs a skill repeatedly, scores against binary evals, mutates the prompt, keeps improvements. |

### Pipeline principles

- **Fresh context per phase** — each phase runs in an isolated subagent. No accumulated drift.
- **No agent judges its own output** — implementer doesn't validate, validator doesn't fix, remediator doesn't verify.
- **Artifacts on disk are the only persistence** — `_relentless_state.md`, `_gates.json`, `_validation_report.json`, `_relentless_config.json`.
- **Circuit breaker** — orchestrator stops after 3 remediation loops if error count isn't decreasing.
- **Fail fast, fail cheap** — preflight catches environment issues in 30 seconds before the pipeline burns budget.

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | Task completed successfully |
| 1 | Max sessions exhausted without completion |
| 2 | Escalation — stuck, looping, or unresolvable blocker |

## Requirements

- Python 3.11+
- [uv](https://github.com/astral-sh/uv)
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI (`claude`)
- Anthropic API key (set `ANTHROPIC_API_KEY` or configure via `claude`)

## License

MIT
