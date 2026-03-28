# Proven

Proven Laws for writing instructions that AI agents follow reliably.

Every law was discovered empirically: run an instruction 5 times with a fresh agent, measure what breaks, rewrite, prove the fix works. Nothing here is theory. Everything is data.

## Laws

| Law | Pattern | Before | After | Cost | Evidence |
|-----|---------|--------|-------|------|----------|
| [LAW-001](methodology/INSTRUCTION-DESIGN.md#law-001-explicit_enumeration) | EXPLICIT_ENUMERATION | 0/5 | 5/5 | $0.79 | [evidence](methodology/evidence/001-explicit-enumeration.md) |
| [LAW-002](methodology/INSTRUCTION-DESIGN.md#law-002-missing_infrastructure) | MISSING_INFRASTRUCTURE | 1/5 | 5/5 | $0.95 | [evidence](methodology/evidence/002-missing-infrastructure.md) |

## How laws are discovered

We use an automated QA loop that follows the [Karpathy autoresearch](https://github.com/karpathy/autoresearch) pattern: one variable (the instruction text), one scalar metric (X/5 compliance), keep/revert branching, runs until convergence.

```
LOOP:
  1. Run instruction 5x with fresh Opus agent
  2. Deterministic judge scores X/5 (no LLM evaluates)
  3. If not 5/5: rewrite instruction, commit
  4. Re-run 5x
  5. Score improved → keep. Otherwise → revert.
  6. GOTO 1
```

The agent that rewrites is not the agent being tested. The judge is code, not an LLM.

## Proven Reliability Index

KDP Machine (first certified playbook):

```
PRI 100 · Opus 4.6 · 27 mars 2026
35/35 runs · 7/7 steps RELIABLE
Coverage: 29% deterministic | 43% structural | 29% manual
```

## What is Proven

Proven builds playbooks that make money, ready to execute by AI agents. The playbooks are free. The intelligence network they feed is the product.

Agents have the tools. They have the orchestration. They don't know what to do. Proven gives them the what — and ensures they do it right.

https://proven.sh
