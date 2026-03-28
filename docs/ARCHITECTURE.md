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
│  Isolated temp directory per run                     │
│  Fresh copy of playbook + state from spec            │
│  All tool calls captured                             │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│                   CLAUDE CODE                         │
│  Fresh instance. No memory. No prior context.         │
│  Reads skill.md, reads state, executes.               │
└──────────────────────┬───────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────┐
│                  TRACE BUILDER                        │
│  Captures all commands and filesystem changes         │
│  Diffs filesystem (snapshot before vs after)          │
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

This runs 5 times per iteration. The Prover reads the verdict, edits skill.md, and triggers the next iteration.

---

## Components

### Test Spec

The spec is the source of truth for what a step SHOULD do. It is read-only — the Prover cannot modify it. Format:

```yaml
step: content_generation
skill: skills/content-generator.md
test_level: DETERMINISTIC          # or STRUCTURAL or MANUAL

timeout_seconds: 600               # per-step, not global
max_budget_usd: 3.0                # Claude API budget per run

initial_state:                     # written to state.json
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
    dest: "books/book-20260327-test/content"

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
    - pattern: "books/*/content/maze_*.png"
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

### Sandbox

Creates an isolated filesystem for each test run.

1. Creates a temporary directory
2. Copies playbook directories (scripts, data, config, templates, skills)
3. Resolves fixtures from either the playbook or shared fixture directories
4. Writes initial state from the spec
5. Takes a filesystem snapshot (all files + hashes) for later diff

Each run gets a completely fresh sandbox. No state leaks between runs.

### Trace Builder

Converts raw execution output into a structured trace.

The trace builder captures every command the agent executes and every file it creates or modifies. It parses commands into structured records (script path + flag/value pairs), diffs the filesystem snapshot to detect all changes, and reads the final state.

Output: a trace containing parsed commands, filesystem diff, and final state.

### Judge

Deterministic assertion engine. Zero LLM involvement.

**Assertion types (executed in order):**

1. **Commands** — For each expected command group: find all actual calls to that script, assert invocation count, match invocations by discriminant, assert each flag matches using matchers
2. **Extra commands** — Any script calls not declared in the spec are flagged
3. **State** — Each expected key/value in final state is checked using matchers
4. **Files created** — Glob for expected file patterns, assert minimum count and optional minimum size
5. **Files not modified** — Read-only files that appear in the diff are flagged
6. **Structural checks** (STRUCTURAL level only) — Per-field constraints: length, range, parity, containment, regex

**Failure classifications:** `FLAG_OMISSION`, `FLAG_WRONG_VALUE`, `FLAG_WRONG_TYPE`, `INVOCATION_COUNT`, `MISSING_INVOCATION`, `EXTRA_COMMANDS`, `STATE_MISSING`, `STATE_WRONG_VALUE`, `MISSING_FILES`, `FILE_TOO_SMALL`, `UNEXPECTED_MODIFICATION`, `STRUCTURAL_CONSTRAINT`

### Runner

Orchestrates N Claude Code subprocess runs.

For each run:
1. Mount a fresh sandbox from the spec
2. Launch Claude Code with a minimal, fixed prompt (skill path + state path)
3. Capture all tool calls from the output
4. Snapshot the filesystem, build the trace
5. Judge the trace against the spec
6. Clean up the sandbox

After N runs, assemble the verdict:
```
5/5 → RELIABLE
2-4/5 → FLAKY
0-1/5 → BROKEN
```

The executor prompt is deliberately minimal — a fresh agent that reads the skill, reads the state, and executes. No extra context, no chain-of-thought guidance.

### Prover

The Karpathy loop agent. A Claude Opus instance that runs autonomously.

**What it CAN modify:** The skill.md file. Nothing else.

**What it CANNOT modify:** Test specs, runner/judge code, scripts, YAML contracts.

**The loop:**
1. Read the verdict
2. Identify which assertions fail most often
3. Edit skill.md with a proving hypothesis
4. Commit
5. Run 5 tests via runner
6. Score improved? Keep. Otherwise revert.
7. If kept: capture the insight (what failed, what fixed it, why)
8. Loop

**Stopping conditions:**
- 5/5 RELIABLE → auto-advance to next broken step
- 10 consecutive reverts → STAGNATION, flag for human review
- All steps RELIABLE → done

**Simplicity criterion:** At equal score, simpler instruction wins. Removing text while maintaining score is a victory.

### Capture Pipeline

Runs after every successful proving iteration (keep). Three outputs:

1. **Structured insight** — before/after diff, failure type, fix type, cost
2. **Standalone markdown** — for marketing and documentation
3. **INSTRUCTION-DESIGN.md update** — appends a new Proven Law with evidence

The capture is triggered by the Prover automatically. No human intervention.

---

## Data Flow

### Certification flow

```
playbook + specs
  → runner (5x Claude Code in sandbox)
    → trace builder (execution output → structured trace)
      → judge (trace vs spec → verdict)
        → PRI calculation (total passed / total runs × 100)
```

### Proving flow

```
verdict (not 5/5)
  → Prover reads failures
    → edits skill.md
      → commit
        → runner (5x)
          → new verdict
            → score improved? keep : revert
              → if kept: capture insight
                → new Proven Law
                → PRI update
```

### Knowledge Flywheel

```
user runs playbook
  → agents call Proven API (scoring, benchmarks)
    → agent executes step
      → metrics reported back
        → aggregated into network intelligence
          → better scores for next user
```

---

## The Three Test Levels

| Level | What it checks | LLM involved? |
|-------|---------------|----------------|
| **DETERMINISTIC** | Exact commands, flags, invocation counts, state values, file existence | No |
| **STRUCTURAL** | Field constraints (length, range, parity, containment) on top of deterministic checks | No |
| **MANUAL** | Human review for subjective quality (content tone, visual layout) | No (human judge) |

All three levels use the same judge infrastructure. DETERMINISTIC and STRUCTURAL are fully automated. MANUAL flags steps for human review but still captures the trace for reproducibility.

---

## Known Limitations (V1)

1. **Tool call capture is partial.** Claude Code's Write/Edit tools bypass bash. The bash wrapper only captures shell commands. Filesystem diff catches the results (files created/modified) but the trace won't show the tool call itself. Command assertions only cover Bash-executed scripts.

2. **Executor prompt is artificially minimal.** Real users give their agents more context. V1 optimizes instructions for a bare prompt. Future: test with multiple prompt styles.

3. **Discriminant field is explicit.** The judge requires a declared discriminant to match expected invocations to actual ones. Implicit matching is a future improvement.

4. **PRI depreciates silently.** The freshness indicator is passive (color changes over time). Future: automated re-certification on model updates.

5. **Single playbook tested.** KDP Machine is the only certified playbook. The methodology's generalizability is assumed, not yet proven across multiple domains.

---

## Separation of Concerns

The Proven engine knows nothing about any specific playbook. It takes a playbook directory, finds specs inside it, and runs. The specs and fixtures live in the playbook — the author knows what "correct" means for their product. The engine provides the testing infrastructure.

The public repository contains the methodology and documentation. Proven Laws are universal — discovered from specific playbooks but applicable to all.
