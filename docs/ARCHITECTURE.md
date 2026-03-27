# Architecture — The Proven QA Loop

> Technical reference for engineers. Assumes you've read [SYSTEM.md](../SYSTEM.md) and understand what the system does. This document explains how.

---

## Overview

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Test Spec   │     │  Skill.md   │     │  Playbook   │
│  (YAML)      │     │  (markdown) │     │  (scripts,  │
│  read-only   │     │  THE only   │     │   data,     │
│  = the judge │     │  mutable    │     │   config)   │
│              │     │  file       │     │  read-only  │
└──────┬───────┘     └──────┬──────┘     └──────┬──────┘
       │                    │                    │
       ▼                    ▼                    ▼
┌──────────────────────────────────────────────────────┐
│                     SANDBOX                           │
│  /tmp/agent-test-{uuid}/                             │
│  Fresh copy of playbook + state from spec            │
│  Stream-json captures all tool calls                 │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│                   CLAUDE CODE                         │
│  claude --print --output-format stream-json           │
│  --model opus --allowedTools "Bash Read Write Edit"   │
│  Fresh instance. No memory. No prior context.         │
│  Reads skill.md, reads state.json, executes.          │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│                  TRACE BUILDER                        │
│  Parses stream-json for Bash tool_use blocks          │
│  Diffs filesystem (snapshot before vs after)          │
│  Reads final state.json                               │
│  Outputs: {commands, files_created, state_after}      │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│                      JUDGE                            │
│  Pure code. No LLM. Deterministic.                    │
│  Compares trace against spec:                         │
│    - Command flags (flag by flag, not regex)           │
│    - Invocation counts                                │
│    - State field values                               │
│    - File existence and size                          │
│    - Extra/unexpected commands                        │
│    - Structural constraints (length, range, parity)   │
│  Output: {passed, failures[], assertions_total}       │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
                   VERDICT
              X/5 = RELIABLE | FLAKY | BROKEN
```

This runs 5 times per iteration. The reformulator reads the verdict, edits skill.md, and triggers the next iteration.

---

## Components

### Test Spec (specs/*.yaml)

The spec is the source of truth for what a step SHOULD do. It is read-only — the reformulator cannot modify it. Format:

```yaml
step: content_generation
skill: skills/content-generator.md
test_level: DETERMINISTIC          # or STRUCTURAL or MANUAL

timeout_seconds: 600               # per-step, not global
max_budget_usd: 3.0                # Claude API budget per run

initial_state:                     # written to .kdp/state.json
  current_book:
    stage: content_generation
    type: maze
    id: "book-20260327-test"
    niche:
      keyword: "dinosaur maze"
      theme: "dinosaur"
      age_range: "4-8"
      marketplace: "fr"

fixtures:                          # files to copy into sandbox
  - source: config/settings.json               # from playbook dir
  - source: "@fixtures/content"                # from shared fixtures
    dest: ".kdp/books/book-20260327-test/content"

expected:
  commands:                        # what scripts should be called
    - script: scripts/maze_generator.py
      count: 3                     # exact number of invocations
      discriminant: --difficulty   # flag that distinguishes calls
      invocations:
        - args:
            --difficulty: "easy"
            --count: "40"
            --output: contains:book-20260327-test/content
            --seed: any_int
            --include-solutions: present

  state_after:                     # what state.json should look like
    current_book.stage: "formatting"
    current_book.content_count: 100

  files_created:                   # what files should exist
    - pattern: ".kdp/books/*/content/maze_*.png"
      min_count: 100

  files_not_modified:              # what should NOT change
    - config/settings.json

  structural_checks:               # STRUCTURAL level only
    - field: current_book.title
      max_length: 200
      must_contain: any_of:["maze", "sudoku"]
    - field: current_book.pages
      min: 20
      max: 500
      even: true
```

**Value matchers** (used in commands and state assertions):

| Matcher | Example | Meaning |
|---------|---------|---------|
| Exact | `"easy"` | String equality |
| `present` | `--include-solutions: present` | Flag exists, any value |
| `any_int` | `--seed: any_int` | Parseable as integer |
| `any_string` | `title: any_string` | Non-empty string |
| `contains:X` | `--output: contains:test/content` | Substring match |
| `starts_with:X` | `id: starts_with:book-` | Prefix match |
| `any_of:[...]` | `type: any_of:["maze","sudoku"]` | Value in list |

**Discriminant field:** When a script is called multiple times, the discriminant is the flag whose value distinguishes one call from another. For maze_generator.py called 3 times, `--difficulty` is the discriminant (easy, medium, hard). The judge matches expected invocations to actual invocations using this field.

### Sandbox (sandbox.py)

Creates an isolated filesystem for each test run.

**mount(spec, playbook_dir) -> sandbox_path:**
1. Creates `/tmp/agent-test-{uuid}/`
2. Copies playbook directories: scripts/, data/, config/, templates/, skills/
3. Copies root files (AGENTS.md, requirements.txt, etc.)
4. Resolves fixtures: `source: "path"` copies from playbook dir, `source: "@fixtures/path"` copies from shared fixtures dir, optional `dest:` overrides placement
5. Creates `.kdp/books/`, `.kdp/sales/`, `.kdp/logs/`
6. Writes `.kdp/state.json` from `spec.initial_state`
7. Creates `.agent-test/` with bash wrapper and empty command log
8. Takes filesystem snapshot (all files + MD5 hashes, excluding `.agent-test/`)

**snapshot(root) -> {path: hash}:** Walks all files, returns relative paths with MD5 hashes. Ignores `.agent-test/` directory.

**diff_snapshots(before, after) -> {created, modified, deleted}:** Set operations on snapshot keys.

**cleanup(sandbox_path):** `shutil.rmtree`.

### Trace Builder (trace_builder.py)

Converts raw data into a structured trace.

**Primary trace source:** Claude Code's `--output-format stream-json --verbose` output. Each assistant message with a `tool_use` block of type `Bash` contains the command string:

```json
{
  "type": "assistant",
  "message": {
    "content": [{
      "type": "tool_use",
      "name": "Bash",
      "input": {"command": "python3 scripts/maze_generator.py --difficulty easy --count 40 ..."}
    }]
  }
}
```

The trace builder parses these into structured records.

**parse_command(raw_cmd) -> {script, args}:** Uses `shlex.split()` for proper quote handling. Extracts script path (tokens containing `scripts/*.py`) and all `--flag value` pairs. Boolean flags (no following value) get `True`.

**build_trace(commands_log, fs_diff, state_after, sandbox_path) -> trace:** Assembles the complete trace from: parsed commands, filesystem diff (created/modified/deleted), and final state.json content.

### Judge (judge.py)

Deterministic assertion engine. Zero LLM involvement.

**Assertion blocks (executed in order):**

1. **Commands** — For each expected command group:
   - Find all actual calls to that script
   - Assert invocation count matches
   - Match invocations by discriminant value
   - Assert each flag exists and matches (using matchers)
   - Classification: `FLAG_OMISSION`, `FLAG_WRONG_VALUE`, `FLAG_WRONG_TYPE`, `INVOCATION_COUNT`, `MISSING_INVOCATION`

2. **Extra commands** — Collect all `scripts/*.py` calls from trace, compare against expected set. Any extras: `EXTRA_COMMANDS`

3. **State** — Flatten `trace.state_after` to dot-notation keys. Compare each expected key/value using matchers. Classification: `STATE_MISSING`, `STATE_WRONG_VALUE`

4. **Files created** — Glob for expected patterns in sandbox. Assert minimum count and optional minimum size. Classification: `MISSING_FILES`, `FILE_TOO_SMALL`

5. **Files not modified** — Check if any read-only file appears in `trace.files_modified`. Classification: `UNEXPECTED_MODIFICATION`

6. **Structural checks** (STRUCTURAL level only) — Per-field constraints: `max_length`, `min_length`, `min`, `max`, `even`, `odd`, `must_contain`, `matches_regex`. Classification: `STRUCTURAL_CONSTRAINT`

**Output:**
```json
{
  "passed": false,
  "failures": [
    {"type": "FLAG_OMISSION", "flag": "--include-solutions", "script": "scripts/maze_generator.py", "discriminant": "--difficulty=easy"}
  ],
  "assertions_total": 18,
  "assertions_passed": 17
}
```

### Runner (runner.py)

Orchestrates N Claude Code subprocess runs.

**run_single(spec, playbook_dir, run_id, model):**
1. Mount sandbox
2. Build executor prompt (fixed, minimal — skill path and state path only)
3. Launch: `claude --print --output-format stream-json --verbose --model {model} --allowedTools "Bash Read Write Edit" --max-budget-usd {spec.max_budget_usd} -p {prompt}`
4. Parse Bash tool calls from stream-json output
5. Snapshot, diff, build trace
6. Judge trace against spec
7. Cleanup sandbox
8. Return judgment with duration and cost metadata

**run_step(spec_path, playbook_dir, runs, model, output_path):**
- Loads spec, runs `run_single` N times, assembles verdict
- Writes verdict.json

**Verdict assembly:**
```
5/5 → RELIABLE
2-4/5 → FLAKY
0-1/5 → BROKEN
```

Classification aggregates failure types across all runs with counts.

**Executor prompt** (fixed, never changes):
```
You are a fresh agent. You have never seen this playbook before.
Read the skill file at {skill_path}. It contains your instructions for this step.
Read the current state at .kdp/state.json.
Execute the step exactly as described in the skill.
When done, do not ask questions. Just stop.
```

### Reformulator (reformulator.md)

The Karpathy loop agent. A Claude Opus instance that runs autonomously.

**What it CAN modify:** The skill.md file. Nothing else.

**What it CANNOT modify:** Test specs, runner/judge code, Python scripts, YAML frontmatter contracts.

**The loop:**
1. Read verdict.json
2. Identify which assertions fail most often
3. Edit skill.md with a reformulation idea
4. git commit
5. Run 5 tests via runner
6. Score improved → keep. Otherwise → `git reset --hard HEAD~1`
7. If kept: run `capture-insight.py` (API + INSTRUCTION-DESIGN.md)
8. Loop

**Stopping conditions:**
- 5/5 RELIABLE → auto-advance to next broken step
- 10 consecutive reverts → STAGNATION, flag for human review
- All steps RELIABLE → log final summary, submit to API, stop

**Simplicity criterion:** At equal score, simpler instruction wins. Removing text while maintaining score is a victory.

### Capture Pipeline (capture-insight.py)

Runs after every successful reformulation (keep). Three outputs:

1. **POST /v1/insights** — structured insight with before/after, failure type, fix type, cost
2. **insights/{step}-{date}.md** — standalone markdown for marketing
3. **methodology/INSTRUCTION-DESIGN.md** — appends new rule with evidence

The capture is triggered by the reformulator as step 9 of its loop. No human intervention.

---

## API

Base URL: `http://localhost:3400` (internal, not yet public)

### Business intelligence

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/score` | POST | Score niche candidates with network data |
| `/v1/benchmarks` | GET | Aggregated sales/pricing data |
| `/v1/metrics` | POST | Submit anonymized performance metrics |
| `/v1/playbook/:id/next` | GET | Next pipeline step |

### Instruction intelligence

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/insights` | POST | Submit a reformulation fix |
| `/v1/insights/design-rules` | GET | Aggregated design rules with evidence counts |
| `/v1/insights/results` | GET | Public-facing results for marketing |

### Certification

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/pri` | POST | Submit a certification snapshot |
| `/v1/pri` | GET | Latest PRI with model, date, freshness |
| `/v1/pri/history` | GET | PRI progression over time |

### Health

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | `{"status": "ok", "version": "0.2.0"}` |

**Data store:** SQLite with WAL mode. Tables: `api_keys`, `metrics`, `keyword_rankings`, `insights`, `design_rules`, `certifications`.

---

## Data Flow

### Certification flow

```
playbook + specs
  → runner (5x Claude Code in sandbox)
    → trace builder (stream-json → structured trace)
      → judge (trace vs spec → verdict)
        → PRI calculation (total passed / total runs × 100)
          → POST /v1/pri
```

### Reformulation flow

```
verdict (not 5/5)
  → reformulator reads failures
    → edits skill.md
      → git commit
        → runner (5x)
          → new verdict
            → score improved? keep : revert
              → if kept: capture-insight.py
                → POST /v1/insights
                → append to INSTRUCTION-DESIGN.md
                → PRI update
```

### Network intelligence flow (future)

```
user runs playbook
  → agents call Proven API (scoring, benchmarks)
    → agent executes step
      → metrics reported back (POST /v1/metrics)
        → aggregated into network intelligence
          → better scores for next user
```

---

## Known Limitations (V1)

1. **Tool call capture is partial.** Claude Code's Write/Edit tools bypass bash. The bash wrapper only captures shell commands. Filesystem diff catches the results (files created/modified) but the trace won't show the tool call itself. Command assertions only cover Bash-executed scripts.

2. **Executor prompt is artificially minimal.** Real users give their agents more context. V1 optimizes instructions for a bare prompt. Future: test with multiple prompt styles.

3. **Discriminant field is explicit.** The judge requires a declared discriminant to match expected invocations to actual ones. Implicit matching is a future improvement.

4. **PRI depreciates silently.** The freshness indicator is passive (color changes over time). Future: automated re-certification on model updates.

5. **Single playbook tested.** KDP Machine is the only certified playbook. The methodology's generalizability is assumed, not yet proven across multiple domains.

---

## File Map

```
proven-monorepo (private)           proven-sh/proven (public)
├── tools/                          ├── methodology/
│   ├── agent-test/                 │   └── INSTRUCTION-DESIGN.md
│   │   ├── sandbox.py              ├── docs/
│   │   ├── trace_builder.py        │   └── ARCHITECTURE.md
│   │   ├── judge.py                ├── SYSTEM.md
│   │   ├── runner.py               ├── README.md
│   │   ├── cli.py                  └── LICENSE
│   │   ├── matchers.py
│   │   ├── capture-insight.py
│   │   ├── reformulator.md
│   │   ├── specs/*.yaml
│   │   └── tests/
│   └── certify/
│       ├── certify.py
│       └── score.py
├── api/
│   └── src/
│       ├── index.ts
│       ├── db.ts
│       └── routes/
│           ├── score.ts
│           ├── benchmarks.ts
│           ├── metrics.ts
│           ├── playbook.ts
│           ├── insights.ts
│           └── pri.ts
├── playbooks/
│   └── kdp-machine/
├── methodology/
├── VISION.md
└── CLAUDE.md
```

The private monorepo contains the tooling, API, and playbook source. The public repo contains the methodology, documentation, and results. The QA loop runs in the private repo and publishes to both.
