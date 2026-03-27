# Instruction Design — Empirical Rules for Agent-Proof Playbooks

> Every rule in this document was discovered by running a playbook instruction 5 times with a fresh Opus agent, measuring what broke, rewriting the instruction, and proving the fix works. Nothing here is theory. Everything is data.

---

## How rules are discovered

Proven uses an automated QA loop (the Karpathy pattern) to test and improve playbook instructions:

1. Run a playbook step 5 times with fresh Claude Opus instances
2. A deterministic judge compares each run against the spec
3. Score: X/5 (how many runs followed the instruction correctly)
4. If not 5/5: rewrite the instruction, re-test, keep or revert
5. When a rewrite fixes the step, extract the design rule

The judge is pure code. No LLM evaluates quality. The metric is binary: did the agent do exactly what the spec says, or didn't it.

Rules are numbered chronologically. Each rule includes the before/after instruction, the failure data, and the cost to discover it.

---

## The rules

### RULE-001: EXPLICIT_ENUMERATION

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

**Failure classification:**
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

For dynamic iteration counts, see RULE-002.

---

### RULE-002: MISSING_INFRASTRUCTURE

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

The upload-prep skill instructed the agent to generate localized packages "for each marketplace in config.marketplaces." The config listed 5 marketplaces: fr, de, it, es, en. But `data/keywords/en.json` and `templates/metadata/description_templates/en.md` did not exist.

The agent generated packages for 4 marketplaces and silently skipped English. This happened in **4 out of 5 runs** — occasionally the agent would create placeholder files on its own, but inconsistently.

**Failure classification:**
- `INVOCATION_COUNT`: expected 5 calls to pdf_assembler.py, got 4
- `INVOCATION_COUNT`: expected 5 calls to cover_builder.py, got 4

#### The fix

The reformulator discovered the root cause was not the instruction wording — it was missing data files. It:

1. Created `data/keywords/en.json` with English Amazon keywords
2. Created `templates/metadata/description_templates/en.md` with English description template
3. Updated the skill to explicitly list all 5 marketplaces with their locales

**Result: 5/5 RELIABLE after infrastructure fix.**

#### The rule

Missing data is an instruction-adjacent failure. The QA loop catches infrastructure gaps, not just wording problems. When an instruction references a config-driven list:

- **DO:** Verify that every entry in the list has its supporting files
- **DO:** Run the QA loop even when you think the instruction is correct — the test may reveal missing infrastructure
- **DON'T:** Assume that "for each X in config" will gracefully handle missing data

**Applies when:**
- A step iterates over a configuration list (marketplaces, locales, content types)
- Each iteration requires data files (templates, keywords, translations)

**Relationship to RULE-001:** RULE-001 handles static iteration (known at design time). RULE-002 handles dynamic iteration (driven by config). Both can co-occur — an instruction may need both explicit enumeration AND infrastructure verification.

---

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

Evidence: RULE-001, 0/5 → 5/5.

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

Evidence: RULE-001, flags omitted in 2/5 runs when listed in prose.

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
| PRI (Proven Reliability Index) | 100.0 | 27 mars 2026 |
| Model | Claude Opus 4.6 | |
| Steps RELIABLE | 7/7 | |
| Steps BROKEN | 0/7 | |
| Design rules discovered | 2 | |
| Average iterations per fix | 1.0 | |
| Average cost per fix | $0.87 | |
| Total reformulation cost | $1.74 | |

---

*This document is auto-enriched by the Proven QA loop. Each successful reformulation appends a new rule with its evidence. The document grows with every fix.*
