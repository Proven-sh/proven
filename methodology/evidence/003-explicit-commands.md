# LAW-003: EXPLICIT_COMMANDS

**In bootstrapping instructions, write explicit commands instead of reading directives. Agents execute commands more reliably than they follow meta-instructions about what to read.**

## Discovery

| | |
|---|---|
| **Discovered** | 28 March 2026 |
| **Playbook** | kdp-machine |
| **Step** | cold-start-status (bootstrapping) |
| **Model** | Claude Sonnet 4.6 |
| **Baseline** | BROKEN 0/3 |
| **After fix** | RELIABLE 5/5 |
| **Iterations** | 3 |
| **Cost** | $0.22 |

## The failure

A fresh Claude Code session was asked: "On en est où avec KDP Machine ?"

The CLAUDE.md said: "Read HANDBOOK.md before responding to any question about this project. Never answer from memory."

The agent ignored this instruction. It answered from memory, launched `git log` and `npm view`, cited obsolete information ("Intelligence Pro $19/mois", "pivot"), and never read `.proven-link.yaml` or `playbooks.yaml`. 3 tests, 3 failures.

The diagnosis: **INSTRUCTION_IGNORED**. The agent treated the reading directive as a suggestion, not a command.

## The before instruction

```markdown
Read HANDBOOK.md before responding to any question about this project. Never answer from memory.
@HANDBOOK.md
```

## The after instruction

```markdown
# KDP Machine

## MANDATORY — read before responding

**Asked about status, PRI, certification?**
```bash
cat .proven-link.yaml
```
Report from this file. Do NOT use git log, npm view, or memory.

**Executing the playbook?** Read AGENTS.md.

**Developing/testing/fixing?** Read HANDBOOK.md.

@HANDBOOK.md
```

## Why it works

The before instruction says "read HANDBOOK.md" — a meta-instruction about what to read. The agent must interpret this, decide when it applies, and choose to follow it. It doesn't.

The after instruction says `cat .proven-link.yaml` — a command. The agent doesn't interpret, it executes. The bash code block is a trigger: "when asked about status, run this command." The anti-pattern is named explicitly: "Do NOT use git log, npm view, or memory."

Three changes made it work:
1. **Explicit command** instead of reading directive (`cat X` not "read X")
2. **Specific to the question type** (status → this command, execution → that file)
3. **Named anti-patterns** ("Do NOT use git log, npm view, or memory")

## The law

**LAW-003: EXPLICIT_COMMANDS** — In bootstrapping instructions (CLAUDE.md, AGENTS.md, entry points), don't write "read file X" or "consult the handbook." Write the exact command: `cat .proven-link.yaml`. Agents execute commands more reliably than they follow instructions about what to read. Name the anti-patterns explicitly.

This is a generalization of LAW-001 (EXPLICIT_ENUMERATION) to the bootstrapping context. LAW-001 says "don't say 'for each X' — write N explicit commands." LAW-003 says "don't say 'read X' — write `cat X`." Same principle: agents follow explicit instructions, not meta-instructions.

## Proof

Verified by the Proving Ground (unified bootstrapping + pipeline testing):

```
=== agent-test: cold-start-status [STRUCTURAL] — 5 runs ===
  run 1/5: PASS
  run 2/5: PASS
  run 3/5: PASS
  run 4/5: PASS
  run 5/5: PASS
  verdict: RELIABLE (5/5)
  Total cost: $0.22
```

First cold start spec certified by the same Proving Ground that certifies pipeline skills.
