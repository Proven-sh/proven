# Proven — How the System Works

> A machine that takes broken instructions, proves they're broken, fixes them, proves the fix works, captures the intelligence, and moves on. Autonomously. For $0.79 per fix.

---

## The problem no one is solving

AI agents are powerful. Orchestration tools exist — Paperclip, Crew AI, Claude Code, Codex. People can set up teams of agents in an afternoon.

But when they give those agents something to do — a playbook, a workflow, a business process — the agents don't follow. They forget parameters. They approximate calculations. They skip steps. They overwrite data. The instruction says A, the agent does something adjacent to A. Sometimes. Other times it does B. Or nothing.

This is not a model capability problem. Claude Opus can reason about quantum physics. It can write a compiler. But ask it to run `maze_generator.py` three times with different difficulty flags, and it will run it once with all difficulties combined. Every time. Five out of five fresh agents made the same mistake.

The problem is not the agent. The problem is the instruction.

---

## Two paths, one choice

When instructions fail, there are two responses:

**Path A: Replace agents with code.** If the agent can't follow, write a Python script that does the job. The agent disappears. The instruction becomes a function call. This works. But you're no longer building playbooks for agents — you're building CLI tools. The value proposition of agent-driven businesses collapses into traditional software.

**Path B: Learn to write instructions agents can follow.** The instruction failed because it was ambiguous, not because the agent was stupid. A step with 8 flags and nested conditionals — no human junior developer would follow that either. The question becomes: how do you design, structure, and formulate instructions so an agent cannot get them wrong?

Proven chose Path B. Not because it's easier — it's harder. But because it's the only path that scales. Every instruction pattern we discover applies to every future playbook. Every fix improves the methodology, not just one script.

---

## The machine

The Proven QA loop follows the [Karpathy autoresearch](https://github.com/karpathy/autoresearch) pattern. One variable changes. One metric is measured. Keep or revert. Loop until convergence.

In autoresearch, the variable is `train.py` and the metric is `val_bpb`. In Proven, the variable is the **instruction text** and the metric is **X/5 compliance** — how many out of 5 fresh agents followed the instruction exactly as specified.

### The cycle

```
1. Fresh sandbox. Clean state. No memory from previous runs.
2. Fresh Opus agent reads the instruction. Executes it.
3. Deterministic judge compares what happened vs what should have happened.
   - Which commands were run? With which flags?
   - Which files were created? Of what size?
   - What does the state look like after?
4. Score: PASS or FAIL. No judgment, no opinion. Code comparison.
5. Repeat 5 times. Score: X/5.
6. If not 5/5: an autonomous reformulator rewrites the instruction.
   - It reads the failure report
   - It modifies the instruction (the ONLY file it can change)
   - It commits
   - It re-runs 5 tests
   - Score improved → keep. Otherwise → revert.
7. Loop until 5/5, stagnation (10 reverts), or manual interrupt.
```

Three critical design constraints make this work:

**The judge is never an LLM.** The evaluation is string comparison, JSON diff, filesystem check. If we used an LLM to evaluate, we'd be using an unreliable tool to measure the reliability of unreliable tools. The snake eating its tail. The judge is pure code.

**The reformulator is not the executor.** The agent that rewrites the instruction is not the agent being tested. The executor is a fresh instance with no context, no memory, no knowledge of why it's being tested. This eliminates the Hawthorne effect — the executor doesn't try harder because it knows it's being watched.

**One variable, one metric.** The instruction text is the only thing that changes between iterations. The scripts, the test specs, the judge, the sandbox — all frozen. When the score improves, we know exactly what caused it: the instruction got better.

---

## The proof

### KDP Machine — first certified playbook

KDP Machine is an autonomous publishing system that generates children's activity books for Amazon KDP. 7 pipeline steps, from niche selection to ads strategy.

**Baseline assessment** (before any fixes):

| Step | Score | Level | Verdict |
|------|-------|-------|---------|
| Niche Scout | 5/5 | MANUAL | RELIABLE |
| Content Generation | 0/5 | DETERMINISTIC | BROKEN |
| Formatting | 5/5 | STRUCTURAL | RELIABLE |
| Cover Design | 5/5 | DETERMINISTIC | RELIABLE |
| Upload Prep | 1/5 | STRUCTURAL | BROKEN |
| Ads Strategy | 5/5 | STRUCTURAL | RELIABLE |
| Performance Analyzer | 5/5 | MANUAL | RELIABLE |

**PRI: 74.3** — 26 out of 35 test runs passing.

Two steps broken. The QA loop went to work.

### Fix #1: Content Generation (0/5 → 5/5)

The instruction told the agent to generate mazes "for each difficulty level." The agent interpreted this as one call with all difficulties combined, instead of three separate calls.

The reformulator identified the failure pattern, rewrote the instruction to use three explicit command blocks (one per difficulty with exact parameter values), committed, re-tested: 5/5.

- **Iterations:** 1
- **Cost:** $0.79
- **Time:** 23 API turns
- **Design rule discovered:** RULE-001 EXPLICIT_ENUMERATION — never say "for each X," give the N commands explicitly.

### Fix #2: Upload Prep (1/5 → 5/5)

The agent generated upload packages for 4 marketplaces instead of 5. Investigation revealed the root cause was not the instruction — it was missing data files. No English keyword file existed, no English description template existed.

The reformulator created the missing files, updated the skill to explicitly list all 5 marketplaces, re-tested: 5/5.

- **Iterations:** 1
- **Cost:** < $1
- **Design insight:** The QA loop catches infrastructure gaps, not just instruction problems.

### Result

**PRI: 74.3 → 88.6 → 100.0**

All 7 steps RELIABLE. 35 out of 35 test runs passing. Total reformulation cost under $2. The entire certification took one session.

---

## The three test levels

Not all steps can be verified to the same depth. Honest certification means stating exactly what was tested and how.

**DETERMINISTIC** (29% of KDP Machine steps) — The judge verifies exact flag values, exact invocation counts, exact state transitions. A wrong flag = FAIL. This is the highest confidence level: if it passes, the instruction is provably correct.

**STRUCTURAL** (43% of steps) — The judge verifies that outputs exist, have the correct format, and respect constraints (length, type, range). It does NOT judge content quality. A title with 50 characters containing the keyword passes — whether it's good prose or terrible. This catches format errors and missing outputs, not creative failures.

**MANUAL** (29% of steps) — The judge can only verify that the agent completed without crashing and that the pipeline state advanced. Niche selection is "viable niche" — that's a judgment call no code can make. 5/5 MANUAL means the agent ran to completion 5 times, not that the output was good 5 times.

The PRI reports all three levels alongside the score:

```
PRI 100 · Opus 4.6 · 27 mars 2026
35/35 runs · 7/7 steps RELIABLE
Coverage: 29% deterministic | 43% structural | 29% manual
```

No pretending to certify what we can't certify.

---

## The Proven Reliability Index

PRI is a single number for the entire playbook:

```
PRI = (total passing runs / total runs) × 100
```

It progresses visibly with each fix. It carries a timestamp, a model version, and a freshness indicator:

- **Green** — tested within the last 30 days
- **Yellow** — 30-90 days since last test
- **Red** — more than 90 days (re-certification required)

Reliability data depreciates with every model update. A PRI certified on Opus 4.6 may not hold on Opus 5. The freshness indicator makes this explicit.

The PRI history tracks progression over time:

```
PRI 74.3 → 88.6 → 100.0
     ↑         ↑        ↑
  baseline   fix #1   fix #2
```

This is the curve. The equivalent of val_bpb dropping in autoresearch. One number, going in the right direction, backed by data.

---

## Certification cost

| Metric | Value |
|---|---|
| Cost per fix | ~$0.87 |
| Average iterations to convergence | 1 |
| Runs per iteration | 5 |
| Cost per full playbook certification (7 steps) | < $2 |
| Time for full certification | < 2 hours |

The entire KDP Machine pipeline went from PRI 74.3 to PRI 100 for $1.74 total. Two fixes, two design rules discovered, zero human intervention during the reformulation loops.
