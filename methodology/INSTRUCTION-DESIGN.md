# Instruction Design — Empirical Laws for Agent-Proof Playbooks

> Every law in this document was discovered by running a playbook instruction 5 times with a fresh Opus agent, measuring what broke, rewriting the instruction, and proving the fix works. Nothing here is theory. Everything is data.

---

## How laws are discovered

Proven uses an automated QA loop (the Karpathy pattern) to test and improve playbook instructions:

1. Run a playbook step 5 times with fresh Claude Opus instances
2. A deterministic judge compares each run against the spec
3. Score: X/5 (how many runs followed the instruction correctly)
4. If not 5/5: rewrite the instruction, re-test, keep or revert
5. When a rewrite fixes the step, extract the Proven Law

The judge is pure code. No LLM evaluates quality. The metric is binary: did the agent do exactly what the spec says, or didn't it.

Laws are numbered chronologically. Each law states the universal principle, then cites the evidence.

---

## The laws

### LAW-001: EXPLICIT_ENUMERATION

**Never say "for each X." Give the N commands explicitly.**

When an instruction requires the same command to run N times with different parameters:

- **DO:** Write N explicit command blocks, each with the exact parameter values
- **DON'T:** Write one command with "for each X" or a parameterized loop
- **WHY:** Agents collapse parameterized instructions into single calls. They optimize for fewer actions, not correctness. Explicit enumeration removes the ambiguity.

**Applies when:**
- A script must be called multiple times with varying arguments
- The number of calls is known at instruction-writing time
- Each call has a different value for at least one parameter

**Does NOT apply when:**
- The number of iterations depends on runtime data
- The iteration count is truly dynamic (see LAW-002)

**Evidence:** KDP Machine, content-generation. 0/5 → 5/5, 1 iteration, $0.79. Agent called a script once instead of 3 times because instruction said "for each difficulty level." Fix: 3 explicit command blocks. [Full case study](evidence/001-explicit-enumeration.md)

---

### LAW-002: MISSING_INFRASTRUCTURE

**When an instruction iterates over a config list, verify that supporting files exist for every entry.**

Missing data is an instruction-adjacent failure. The QA loop catches infrastructure gaps, not just wording problems.

- **DO:** Verify that every entry in a config-driven list has its supporting files before certifying
- **DO:** Run the QA loop even when you think the instruction is correct — the test may reveal missing infrastructure
- **DON'T:** Assume that "for each X in config" will gracefully handle missing data
- **WHY:** Agents silently skip iterations when data files are absent. They don't error — they just produce incomplete output. This is harder to detect than a crash.

**Applies when:**
- A step iterates over a configuration list (marketplaces, locales, content types)
- Each iteration requires data files (templates, keywords, translations)

**Relationship to LAW-001:** LAW-001 handles static iteration (known at design time). LAW-002 handles dynamic iteration (driven by config). Both can co-occur.

**Evidence:** KDP Machine, upload-prep. 1/5 → 5/5, 1 iteration, $0.95. Agent generated 4/5 marketplace packages because English data files were missing. Fix: created missing files + explicit marketplace list. [Full case study](evidence/002-missing-infrastructure.md)

---

## Anti-patterns

Patterns that consistently cause agent failures, derived from QA loop data.

### ANTI-001: Implicit iteration

```
# BAD — agent will collapse to 1 call
Run the script for each difficulty level (easy, medium, hard)
with counts proportional to the distribution.

# GOOD — agent runs exactly 3 commands
Run the script 3 times:
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
python3 scripts/my_script.py \
  --difficulty easy \
  --count 40 \
  --output output/ \
  --include-solutions \
  --seed 42 \
  --theme test \
  --shape rectangular
```

Evidence: LAW-001, flags omitted in 2/5 runs when listed in prose.

### ANTI-003: Silent config assumptions

```
# BAD — agent skips entries with missing data
Process each marketplace in config.marketplaces

# GOOD — verify data exists, fail explicitly
Process these 5 marketplaces: fr, de, it, es, en
For each, verify these files exist before proceeding:
  - data/keywords/{locale}.json
  - templates/descriptions/{locale}.md
If any file is missing, STOP and report the error.
```

Evidence: LAW-002, 1/5 → 5/5.

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
| Proven Laws discovered | 2 | |
| Average iterations per fix | 1.0 | |
| Average cost per fix | $0.87 | |
| Total proving cost | $1.74 | |

---

*This document is auto-enriched by the Proven QA loop. Each successful proving iteration appends a new law with its evidence. The document grows with every fix.*
