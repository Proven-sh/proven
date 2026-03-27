# RULE-001: EXPLICIT_ENUMERATION — Case Study

**Playbook:** KDP Machine (Amazon KDP activity book publisher)
**Step:** content-generation
**Date:** 27 mars 2026
**Model:** Claude Opus 4.6

## The instruction (before)

The content-generation skill told the agent to generate maze images at three difficulty levels:

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

## The failure

The agent interpreted this as ONE call with `--difficulty easy,medium,hard` and `--count 100` (the total). It should have been THREE separate calls: easy (40), medium (40), hard (20).

This failed **5 out of 5 runs** with fresh independent agents. The failure was not random — every agent resolved the ambiguity the same wrong way.

**Judge output:**
- `INVOCATION_COUNT`: expected 3 calls to maze_generator.py, got 1
- `STATE_WRONG_VALUE`: content_count expected 100, got 200

## The instruction (after)

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

## Result

**5/5 RELIABLE** on first attempt after rewrite. Zero iterations wasted.

## Cost

$0.79 total (23 API turns for the reformulator + 5 executor runs).
