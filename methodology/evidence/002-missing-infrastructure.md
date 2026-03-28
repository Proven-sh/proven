# LAW-002: MISSING_INFRASTRUCTURE — Case Study

**Playbook:** KDP Machine (Amazon KDP activity book publisher)
**Step:** upload-prep
**Date:** 27 mars 2026
**Model:** Claude Opus 4.6

## The instruction (before)

The upload-prep skill told the agent to generate localized packages for all configured marketplaces. The config listed 5 marketplaces: fr, de, it, es, en.

The instruction said "for each marketplace in config.marketplaces" and listed reference tables for FR, DE, IT, ES — but not EN.

## The failure

The agent generated packages for 4 marketplaces and silently skipped English. No error, no warning — just incomplete output.

Root cause: `data/keywords/en.json` and `templates/metadata/description_templates/en.md` did not exist. The agent couldn't generate an English package because the source data wasn't there.

This happened in **4 out of 5 runs**. In the 5th run, the agent happened to create placeholder files on its own — inconsistent behavior.

**Judge output:**
- `INVOCATION_COUNT`: expected 5 calls to pdf_assembler.py, got 4
- `INVOCATION_COUNT`: expected 5 calls to cover_builder.py, got 4

## The fix

The Prover discovered the root cause was not the instruction wording — it was missing data files. It:

1. Created `data/keywords/en.json` with English Amazon keywords
2. Created `templates/metadata/description_templates/en.md` with English description template
3. Updated the skill to explicitly list all 5 marketplaces with their locales and domains

## Result

**5/5 RELIABLE** after infrastructure fix.

## Cost

$0.95 total (28 API turns for the Prover + 5 executor runs).

## Insight

The QA loop revealed an infrastructure gap, not an instruction design flaw. The instruction was reasonable ("for each marketplace in config"), but the supporting files were incomplete. This is a category of failure that code review wouldn't catch — you have to actually run the instruction to discover it.
