---
name: relentless-preflight
description: Pre-flight environment check before running the relentless pipeline. Validates dependencies, Databricks profile, ports, PRD structure, and writes _relentless_config.json for downstream phases.
user-invocable: true
---

# Pre-flight Check

Validate the environment before the relentless pipeline starts. Catches config, dependency, and permission issues upfront — before they waste budget 3 sessions deep.

**Announce at start:** "Running pre-flight checks."

## Pipeline Position

```
relentless-preflight → _relentless_config.json (THIS SKILL — runs BEFORE pipeline)
    ↓
relentless-prd → prd_<slug>.md
    ↓
relentless-gates → failing gate tests + _gates.json
    ↓
relentless-implement → makes gate tests pass
    ↓
relentless-validate → live execution + error capture
    ↓
relentless-remediate → targeted fixes
    ↓
relentless-orchestrate → loop until clean
```

## Why This Exists

Without a pre-flight check, the pipeline discovers environment issues reactively:
- Session 1 of implementation fails because `uv` isn't installed → wasted budget
- Validator hangs because Databricks profile doesn't exist → wasted budget + manual intervention
- Remediator can't connect to database because port is occupied → wasted loop

Pre-flight checks are cheap (< 30 seconds, zero API cost). Pipeline failures are expensive (minutes of compute, multiple sessions, context budget). The ROI is obvious.

## Process

### Step 1: Detect project stack

Read the PRD (if provided) or infer from the project structure:

```bash
# Check for PRD
ls prd_*.md 2>/dev/null

# Infer stack from directory structure
ls -d backend/ frontend/ 2>/dev/null
ls pyproject.toml package.json Makefile databricks.yml bundle.yml app.yaml 2>/dev/null
```

Determine:
- **Stack**: `backend`, `frontend`, `fullstack`, or `unknown`
- **Backend framework**: FastAPI (uvicorn), Flask, Django, or none
- **Frontend framework**: React (vite/CRA), Next.js, or none
- **Databricks integration**: Yes (databricks.yml/bundle.yml present) or No
- **PRD file**: Path to `prd_*.md` if found

### Step 2: Run checks

Run all applicable checks. Each check produces: `PASS`, `WARN`, or `FAIL`.

#### Core checks (always run)

| Check | Command | PASS | WARN | FAIL |
|-------|---------|------|------|------|
| Git clean | `git status --porcelain` | Clean or only untracked | Uncommitted changes | Not a git repo |
| Git branch | `git branch --show-current` | Any branch | — | Detached HEAD |
| Claude CLI | `which claude` | Found | — | Not found |
| relentless CLI | `which relentless` | Found | — | Not found |

#### Backend checks (if backend detected)

| Check | Command | PASS | WARN | FAIL |
|-------|---------|------|------|------|
| Python (uv) | `uv run python --version` | >=3.10 | — | Not found or <3.10 |
| Dependencies | `cd backend && uv sync --dry-run 2>&1` | All resolved | Warnings | Resolution failures |
| Port 8000 | `lsof -ti:8000` | Free | — | In use (show PID) |
| Backend tests exist | `ls backend/tests/ 2>/dev/null` | Tests found | Empty | No tests dir |

#### Frontend checks (if frontend detected)

| Check | Command | PASS | WARN | FAIL |
|-------|---------|------|------|------|
| Node.js | `node --version` | >=18 | >=16 <18 | Not found or <16 |
| npm/pnpm | `npm --version` or `pnpm --version` | Found | — | Not found |
| Dependencies | `cd frontend && npm ls --depth=0 2>&1` | Installed | Missing some | node_modules missing |
| Port 5173 | `lsof -ti:5173` | Free | — | In use (show PID) |

#### Databricks checks (if Databricks integration detected)

| Check | Command | PASS | WARN | FAIL |
|-------|---------|------|------|------|
| Databricks CLI | `databricks --version` | Found | — | Not found |
| Profile exists | `grep '^\[' ~/.databrickscfg` | Profile found | Multiple profiles (ask user) | No config file |
| Profile connects | `databricks workspace list / --profile <profile> 2>&1 \| head -3` | Lists files | Slow (>5s) | Auth error |
| Bundle valid | `databricks bundle validate --profile <profile> 2>&1` | Valid | Warnings | Errors |
| App name | Parse from `app.yaml` or `databricks.yml` | Found | — | Not found (if app deployment expected) |

#### PRD checks (if PRD file found)

| Check | Command | PASS | WARN | FAIL |
|-------|---------|------|------|------|
| Agent Handoff present | Grep for `## Agent Handoff` in PRD | Found | — | Missing |
| Acceptance criteria | Grep for `AC-` patterns | Found (with count) | <3 ACs | None found |
| Stack field | Parse `stack` from handoff JSON | Matches detected stack | Mismatch | Missing |
| Validation surfaces | Parse `validation_surfaces` | Defined | — | Missing (warn: validator will infer) |

### Step 3: Resolve Databricks profile

If Databricks integration is detected, resolve the profile deterministically:

1. **PRD Agent Handoff** specifies `databricks_profile` → use it
2. **`databricks.yml` / `bundle.yml`** references a profile → use it
3. **Only one profile in `~/.databrickscfg`** → use it (with confirmation)
4. **Multiple profiles, none specified** → ask user: "Which Databricks CLI profile? Available: [list]"
5. **No profiles** → FAIL: "No Databricks CLI profiles configured. Run `databricks configure` first."

### Step 4: Write `_relentless_config.json`

Write the resolved configuration for all downstream phases:

```json
{
  "created_at": "<ISO timestamp>",
  "created_by": "relentless-preflight",
  "project_root": "/absolute/path/to/project",
  "prd_file": "prd_<slug>.md",
  "stack": "fullstack",
  "backend": {
    "framework": "fastapi",
    "port": 8000,
    "test_command": "cd backend && uv run pytest tests/gates/ --no-header -q --timeout=30",
    "start_command": "cd backend && uv run uvicorn app.main:app --reload --port 8000"
  },
  "frontend": {
    "framework": "react-vite",
    "port": 5173,
    "start_command": "cd frontend && npm run dev"
  },
  "databricks": {
    "profile": "fe-westus",
    "app_name": "lakemeter-app",
    "bundle_config": "databricks.yml",
    "deploy_command": "databricks bundle deploy --profile fe-westus",
    "run_command": "databricks bundle run --profile fe-westus"
  },
  "pipeline": {
    "gates":      { "sessions": 2, "timeout_min": 15 },
    "implement":  { "sessions": 5, "timeout_min": 30 },
    "validate":   { "sessions": 2, "timeout_min": 20 },
    "revalidate": { "sessions": 1, "timeout_min": 15 },
    "remediate_max_loops": 3,
    "implement_max_retries": 2,
    "stuck_threshold": 2,
    "tool_repeat_limit": 3,
    "max_tool_calls": 200
  },
  "preflight": {
    "checks_run": 14,
    "passed": 12,
    "warnings": 1,
    "failures": 1,
    "blocking_failures": ["databricks_profile_connect"]
  }
}
```

**Fields are nullable.** If no frontend exists, `frontend` is `null`. If no Databricks integration, `databricks` is `null`. Downstream phases check for null before using a section.

### Step 5: Report results

Present a human-readable summary:

```
## Pre-flight Results

### Environment
- Project: /path/to/project
- Stack: fullstack (FastAPI + React/Vite)
- Databricks: fe-westus profile
- PRD: prd_demand-plan.md (15 ACs, 12 machine)

### Checks
| # | Check | Status | Detail |
|---|-------|--------|--------|
| 1 | Git clean | PASS | On branch feat/demand-plan |
| 2 | Python (uv) | PASS | 3.12.1 |
| 3 | Node.js | PASS | 20.11.0 |
| 4 | Backend deps | PASS | All resolved |
| 5 | Frontend deps | WARN | 2 peer dependency warnings |
| 6 | Port 8000 | PASS | Free |
| 7 | Port 5173 | PASS | Free |
| 8 | Databricks CLI | PASS | 0.234.0 |
| 9 | Profile connects | PASS | fe-westus workspace accessible |
| 10 | Bundle valid | PASS | No errors |
| 11 | PRD structure | PASS | Agent Handoff present |

### Verdict
All critical checks passed. 1 warning (peer deps — non-blocking).
Config written to: _relentless_config.json

Ready to start pipeline.
```

If there are FAIL checks:

```
### Verdict
2 blocking failures — pipeline cannot start.

FAIL: Databricks profile 'fe-prod' cannot connect (403 Forbidden)
  → Fix: Check token expiry. Run `databricks configure --profile fe-prod`

FAIL: Port 8000 in use (PID 12345: python)
  → Fix: Kill process or use a different port

Fix these issues and re-run /relentless-preflight.
```

### Step 6: Gate decision

- **Zero FAILs** → Config written. Tell user/orchestrator: "Pre-flight passed. Ready to start pipeline."
- **Any FAILs** → Do NOT write `_relentless_config.json`. Report failures with fix suggestions. The pipeline should not start until pre-flight passes.
- **Only WARNs** → Write config. Pipeline can start, but note warnings.

## How Downstream Phases Use `_relentless_config.json`

| Phase | What it reads |
|-------|--------------|
| **Orchestrator** | `project_root`, `prd_file`, `databricks.profile` (passes to all subagents) |
| **Implementer** | `backend.test_command` (to verify gates), `stack` (to scope work) |
| **Validator** | `databricks.profile`, `backend.port`, `frontend.port`, `databricks.app_name`, `stack` |
| **Remediator** | `project_root` (to resolve `source_file` paths), `databricks.profile` (if config fixes needed) |

Each phase reads the file at startup. If `_relentless_config.json` is missing, phases fall back to their existing inference logic (read PRD, check project structure, ask user). The config file is an optimization, not a hard dependency.

## Design Principles

### Fail fast, fail cheap

A 30-second pre-flight that catches a missing Databricks profile saves 10 minutes of pipeline execution that would eventually fail anyway. Pre-flight is the cheapest insurance in the pipeline.

### Config file is a contract, not a cache

`_relentless_config.json` isn't just "what pre-flight found" — it's a contract between phases. The orchestrator writes it once, every phase reads it, and the values are authoritative. No phase should override config values without updating the file.

### Non-blocking by default

Pre-flight is advisory for the orchestrator. If the orchestrator is invoked without running pre-flight first, phases fall back to their existing inference. Pre-flight improves reliability, it doesn't gate it.

### Checks are idempotent

Running pre-flight twice produces the same result (assuming no environment changes). Safe to re-run after fixing failures.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Pre-flight hangs on Databricks check | Profile token may be expired. Run `databricks configure --profile <name>` |
| Port in use but no visible process | Check for zombie processes: `lsof -ti:<port>` then `kill -9 <PID>` |
| `uv` not found | Install: `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| `databricks` CLI not found | Install: `curl -fsSL https://raw.githubusercontent.com/databricks/setup-cli/main/install.sh \| sh` |
| PRD has no Agent Handoff | Warn but don't block — validator will infer surfaces from stack |
| Multiple PRD files found | Ask user which one to use, or use the most recently modified |
