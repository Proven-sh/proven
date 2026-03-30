# Instruction Design — Proven Laws for Agent-Proof Playbooks

> Every rule in this document was discovered by running a playbook instruction 5 times with a fresh Opus agent, measuring what broke, rewriting the instruction, and proving the fix works. Nothing here is theory. Everything is data.

---

## How laws are discovered

Proven uses an automated QA loop (the Karpathy pattern) to test and improve playbook instructions:

1. Run a playbook step 5 times with fresh Claude Opus instances
2. A deterministic judge compares each run against the spec
3. Score: X/5 (how many runs followed the instruction correctly)
4. If not 5/5: rewrite the instruction, re-test, keep or revert
5. When a rewrite fixes the step, extract the Proven Law

The judge is pure code. No LLM evaluates quality. The metric is binary: did the agent do exactly what the spec says, or didn't it.

Laws are numbered chronologically. Each law includes the before/after instruction, the failure data, and the cost to discover it.

---

## The laws

### LAW-001: EXPLICIT_ENUMERATION

**Never say "for each X." Give the N commands explicitly.**

| | |
|---|---|
| **Discovered** | 27 mars 2026 |
| **Playbook** | KDP Machine |
| **Step** | content-generation |
| **Model** | Claude Opus 4.6 |
| **Baseline** | 0/5 BROKEN |
| **After fix** | 5/5 RELIABLE |
| **Iterations** | 1 |
| **Cost** | $0.79 |

#### The failure

The content-generation skill instructed the agent to generate maze images across three difficulty levels. The instruction said:

```
Build the difficulty list from maze.difficulty_distribution keys.

For a single-shape book:

python3 scripts/maze_generator.py \
  --shape {shape} \
  --difficulty easy,medium,hard \
  --count {default_activity_count} \
  --output .kdp/books/{book_id}/content/ \
  --include-solutions \
  --seed {SEED} \
  --theme {niche.theme}
```

The agent interpreted this as ONE call with `--difficulty easy,medium,hard` and `--count 100` (the total). It should have been THREE separate calls: easy (40), medium (40), hard (20).

This failed **100% of the time** — 0 out of 5 runs, across 5 completely independent agents. The failure was not random. The instruction was ambiguous and every agent resolved the ambiguity the same wrong way.

**Diagnosis:**
- `INVOCATION_COUNT`: expected 3 calls to maze_generator.py, got 1
- `STATE_WRONG_VALUE`: content_count expected 100, got 200 (agent counted all files including solutions)

#### The fix

Replace the single parameterized command with three explicit command blocks:

```
Step 6c — Run maze_generator.py once PER difficulty (3 separate commands).

You MUST run 3 separate commands — one for each difficulty.
Do NOT combine difficulties into a single call.

Command 1 — easy:
python3 scripts/maze_generator.py \
  --difficulty easy \
  --count 40 \
  --output .kdp/books/{book_id}/content/ \
  --include-solutions \
  --seed {SEED} \
  --theme {niche.theme} \
  --shape {shape}

Command 2 — medium:
python3 scripts/maze_generator.py \
  --difficulty medium \
  --count 40 \
  --output .kdp/books/{book_id}/content/ \
  --include-solutions \
  --seed {SEED} \
  --theme {niche.theme} \
  --shape {shape}

Command 3 — hard:
python3 scripts/maze_generator.py \
  --difficulty hard \
  --count 20 \
  --output .kdp/books/{book_id}/content/ \
  --include-solutions \
  --seed {SEED} \
  --theme {niche.theme} \
  --shape {shape}
```

**Result: 5/5 RELIABLE on first attempt after rewrite.** Zero iterations wasted.

#### The rule

When an instruction requires the same command to run N times with different parameters:

- **DO:** Write N explicit command blocks, each with the exact parameter values
- **DON'T:** Write one command with "for each X" or a parameterized loop
- **WHY:** Agents collapse parameterized instructions into single calls. They optimize for fewer actions, not correctness. Explicit enumeration removes the ambiguity.

**Applies when:**
- A script must be called multiple times with varying arguments
- The number of calls is known at instruction-writing time
- Each call has a different value for at least one parameter

**Does NOT apply when:**
- The number of iterations depends on runtime data (e.g., "for each marketplace in config")
- The iteration count is truly dynamic

For dynamic iteration counts, see LAW-002 (pending).

---


### LAW-002: MISSING_INFRASTRUCTURE

**When an instruction iterates over a config list, verify that supporting files exist for every entry.**

| | |
|---|---|
| **Discovered** | 27 mars 2026 |
| **Playbook** | KDP Machine |
| **Step** | upload-prep |
| **Model** | Claude Opus 4.6 |
| **Baseline** | 1/5 BROKEN |
| **After fix** | 5/5 RELIABLE |
| **Iterations** | 1 |
| **Cost** | $0.95 |

#### The failure

The upload-prep skill iterated over marketplaces from config. English (EN) marketplace data files were missing. The agent silently produced 4/5 marketplace packages — it didn't error, it just skipped the iteration with missing data.

**Diagnosis:**
- `MISSING_FILES`: expected 5 marketplace packages, got 4

#### The fix

1. Created missing English data files (keywords, templates)
2. Added explicit marketplace list in the skill instruction

**Result: 5/5 RELIABLE on first attempt after creating infrastructure.**

#### The rule

Missing data is an instruction-adjacent failure. The QA loop catches infrastructure gaps, not just wording problems.

- **DO:** Verify that every entry in a config-driven list has its supporting files before certifying
- **DO:** Run the QA loop even when you think the instruction is correct — the test may reveal missing infrastructure
- **DON'T:** Assume that "for each X in config" will gracefully handle missing data
- **WHY:** Agents silently skip iterations when data files are absent. They don't error — they just produce incomplete output. This is harder to detect than a crash.

**Applies when:**
- A step iterates over a configuration list (marketplaces, locales, content types)
- Each iteration requires data files (templates, keywords, translations)

**Relationship to LAW-001:** LAW-001 handles static iteration (known at design time). LAW-002 handles dynamic iteration (driven by config). Both can co-occur.

See [evidence/002-missing-infrastructure.md](evidence/002-missing-infrastructure.md) for full case study.


### LAW-003: EXPLICIT_COMMANDS

**In bootstrapping instructions, write explicit commands (`cat .proven-link.yaml`) instead of reading directives ("read HANDBOOK.md"). Agents execute commands more reliably than they follow meta-instructions about what to read. Name anti-patterns explicitly.**

| | |
|---|---|
| **Discovered** | 28 March 2026 |
| **Playbook** | kdp-machine |
| **Step** | cold-start-status (bootstrapping) |
| **Model** | Claude Sonnet 4.6 |
| **Baseline** | 0/3 |
| **After fix** | 5/5 (RELIABLE) |
| **Iterations** | 3 |
| **Cost** | $0.22 |

#### The failure

Agent ignored CLAUDE.md reading directives ("Read HANDBOOK.md before responding"). Answered from memory using `git log` and `npm view`. Never read `.proven-link.yaml`. Cited obsolete information. BROKEN 0/3.

#### The fix

Before: `Read HANDBOOK.md before responding. Never answer from memory. @HANDBOOK.md`

After:
```markdown
**Asked about status, PRI, certification?**
```bash
cat .proven-link.yaml
```
Report from this file. Do NOT use git log, npm view, or memory.
```

Three changes: explicit command (`cat X`), specific to question type, named anti-patterns.

#### The rule

Generalization of LAW-001 (EXPLICIT_ENUMERATION) to bootstrapping. LAW-001: "don't say 'for each X' — write N commands." LAW-003: "don't say 'read X' — write `cat X`." Same principle at a different level. First law discovered and certified via cold start spec (unified bootstrapping + pipeline testing).

See [evidence/003-explicit-commands.md](evidence/003-explicit-commands.md) for full case study.
| | |
|---|---|
| **Discovered** | 29 March 2026 |
| **Playbook** | kdp-machine |
| **Step** | cover_design |
| **Model** | Claude Opus 4.6 |
| **Baseline** | 5/5 |
| **After fix** | 5/5 |
| **Iterations** | 6 |
| **Cost** | $0.00 |

#### The failure



#### The fix

See git diff for details.

#### The rule

simplify cover-designer: flatten branching, reduce to 6 steps

## Anti-patterns

Patterns that consistently cause agent failures, derived from QA loop data.

### ANTI-001: Implicit iteration

```
# BAD — agent will collapse to 1 call
Run maze_generator.py for each difficulty level (easy, medium, hard)
with counts proportional to the distribution.

# GOOD — agent runs exactly 3 commands
Run maze_generator.py 3 times:
Command 1: --difficulty easy --count 40
Command 2: --difficulty medium --count 40
Command 3: --difficulty hard --count 20
```

Evidence: LAW-001, 0/5 → 5/5.

### ANTI-002: Flags listed in prose

```
# BAD — agent forgets flags
Run the script with --difficulty, --count, --output, --seed,
--theme, --shape, --include-solutions flags.

# GOOD — agent copies the command
Run this exact command:
python3 scripts/maze_generator.py \
  --difficulty easy \
  --count 40 \
  --output .kdp/books/{book_id}/content/ \
  --include-solutions \
  --seed {SEED} \
  --theme {niche.theme} \
  --shape {shape}
```

Evidence: LAW-001, flags omitted in 2/5 runs when listed in prose.

---

## Test levels

Not all steps can be tested to the same depth. Every step declares its level:

| Level | What the judge verifies | Confidence |
|---|---|---|
| **DETERMINISTIC** | Exact flags, exact values, exact file counts, exact state | High — the judge says PASS/FAIL with certainty |
| **STRUCTURAL** | Output exists, correct format, respects constraints (length, type, range) | Medium — format is correct, content quality is not judged |
| **MANUAL** | Agent completed without crashing, state advanced | Low — the agent did something, but we can't verify what |

A 5/5 DETERMINISTIC means the instruction is proven correct. A 5/5 MANUAL means it doesn't crash. Different confidence levels, both useful, honestly reported.

---

## Metrics

| Metric | Value | Date |
|---|---|---|
| PRI (Proven Reliability Index) | 88.6 | 27 mars 2026 |
| Model | Claude Opus 4.6 | |
| Steps RELIABLE | 6/7 | |
| Steps BROKEN | 1/7 (upload-prep) | |
| Proven Laws discovered | 1 | |
| Average iterations per fix | 1.0 | |
| Average cost per fix | $0.79 | |
| Total proving cost | $0.79 | |

---

*This document is auto-enriched by the Proven QA loop. Each successful proof appends a new law with its evidence. The document grows with every fix.*
