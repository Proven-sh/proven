# Proven — Component Status

Auto-generated from CIRCUIT.md on 2026-03-30T12:24:39Z.

| Component | Status | Source |
|---|---|---|
| Skill | Markdown instruction an agent follows for one pipeline stage | Law (applies known laws), Author (writes) | Spec (declares expected behavior) | LIVE | playbook/skills/*.md |
| Spec | YAML declaring expected behavior: commands, state, files, forbidden commands, response content. Pipeline specs use `initial_state`, cold start specs use `prompt`. | Skill (spec tests what skill declares) | Sandbox (configures environment) | LIVE | playbook/specs/*.yaml |
| Runner | Orchestrates the test: loads spec, creates sandbox, launches agent, captures trace, invokes judge. Popen + start_new_session for safe cleanup | Spec (test definition), Sandbox (environment) | Trace (captures execution), Judge (passes results) | LIVE | tools/agent-test/runner.py |
| Sandbox | Isolated filesystem per test run, fresh copy each time | Spec (initial state, fixtures) | Trace (agent runs inside sandbox, trace records what happens) | LIVE | tools/agent-test/sandbox.py |
| Trace | Structured record of what the agent did: commands, files, state | Sandbox (agent execution captured) | Judge (reads trace as "actual") | LIVE | tools/agent-test/trace_builder.py |
| Judge | Pure code, zero LLM. Compares spec (expected) to trace (actual) | Spec (expected), Trace (actual) | Verdict (produces classified result) | LIVE | tools/agent-test/judge.py |
| Verdict | Result of N runs: RELIABLE 5/5, FLAKY 3-4/5, BROKEN <=2/5 | Judge (per-run results aggregated) | Prover (reads verdict to know what broke) | LIVE | verdict-*.json |
| Prover | Single-shot editor: reads verdict + spec + laws, edits ONE mutable file, exits. cli.py drives the prove loop (edit → verify → test → keep/revert). Post-edit git verification ensures only the declared mutable file is changed and frontmatter is intact. | Verdict (what broke), Law (known fixes) | Skill or CLAUDE.md (edits the instruction), Capture (triggers on success) | LIVE | tools/agent-test/prover.md (rename from reformulator.md) |
| Capture (-> API) | POST /v1/insights: pushes fix data to instruction reliability DB | Prover (successful fix) | API (stores insight data) | LIVE | tools/agent-test/capture-insight.py |
| Capture (-> INSTRUCTION-DESIGN append) | Appends new law to methodology/INSTRUCTION-DESIGN.md | Prover (successful fix) | Law (new law indexed) | LIVE | tools/agent-test/capture-insight.py |
| Capture (-> evidence/ standalone) | Generates standalone case study in methodology/evidence/ | Prover (successful fix) | Marketing (publishable proof) | DESIGNED | -- |
| Law | Universal principle discovered empirically. Proven by data. | Capture (extracts and generalizes) | Skill (authors and Prover apply laws to new instructions) | LIVE | methodology/INSTRUCTION-DESIGN.md |
| Pipeline | Linear state machine connecting skills. Stages flow one direction, no branching. | Playbook author (designs stages) | Orchestrator (reads pipeline to dispatch) | LIVE | playbook/playbook.yaml |
| State | Runtime position: current book, current stage, retry count | Pipeline (defines valid stages), Skill (reads/writes state) | Orchestrator (checks current stage), Skill (reads to decide what to do) | LIVE | playbook/.kdp/state.json |
| Orchestrator | Dispatches the correct skill based on current state and pipeline | Pipeline (stage order), State (current position) | Skill (invokes the next skill to execute) | LIVE | playbook/skills/kdp-orchestrator.md |
| Skill | Markdown instruction the agent follows for one stage | Orchestrator (dispatched), Law (design principles applied) | Script (invokes deterministic tools), State (writes progress) | LIVE | playbook/skills/*.md |
| Script | Deterministic Python CLI tool. The muscle: computation, not decision. | Skill (invokes with CLI args) | Files (produces output), stdout (returns JSON) | LIVE | playbook/scripts/*.py |
| Scorer: command/state/files | Evaluates command compliance, state transitions, file outputs | Rubric (defines checks), Trace (provides actual data) | Judge (dispatches scorers, aggregates results) | LIVE | tools/agent-test/judge.py |
| Scorer: visual | Evaluates image dimensions, DPI, visual properties via Gemini Flash Vision | Rubric (defines visual quality checks) | Judge (aggregates visual scores) | LIVE | tools/agent-test/scorers/visual.py |
| Scorer Registry | Plugin mechanism for registering and discovering scorers | Scorer implementations (register themselves) | Judge (discovers available scorers) | LIVE | tools/agent-test/scorers/registry.py |
| Benchmark Provider: book-cover | Analyzes top competitor covers to define "what good looks like" | Market data (competitor covers) | Calibration (produces reference dataset), Design Brief (actionable guidance) | IMPLEMENTED | kdp-machine/scripts/benchmark_covers.py (playbook-level, not engine) |
| Calibration: book-cover (code) | Code to produce reference data from benchmark provider | Benchmark Provider (fetches raw data) | Scorer (anchors scores to market reality) | IMPLEMENTED | kdp-machine/specs/calibration/ |
| Calibration: book-cover (data) | Actual reference dataset for book cover scoring | Calibration code (produces the dataset) | Scorer (consumes reference data for scoring) | LIVE | kdp-machine/specs/calibration/book-cover/ (v2, source: amazon.fr) |
| Benchmark Provider: Amazon performance | Fetches Amazon sales/ranking data for niche calibration | Market data (Amazon API) | Calibration (produces performance reference data) | PLANNED | -- |
| Quality loop (--quality flag) | CLI flag to run quality scoring alongside reliability testing | Runner (integrates quality scoring into test runs) | Verdict (adds quality dimensions to results) | LIVE | tools/agent-test/runner.py, tools/certify/certify.py |
| Palette Extractor | Extracts primary/accent/highlight colors from an image via Pillow quantization | Visual Pipeline (winner image) | Visual Identity (palette field) | LIVE | tools/visual/palette.py |
| Image Generator | Provider-agnostic AI image generation (Gemini Flash, placeholder) | Playbook skill (prompt + design brief) | Visual Pipeline (variant images) | LIVE | tools/visual/generate.py |
| Visual Pipeline | Generate-and-score loop: generate N variants, score each, select best, retry with feedback | Image Generator, Visual Scorer | Visual Identity JSON + winner image | LIVE | tools/visual/pipeline.py |
| Visual Identity | JSON contract: winner image + score + palette + generation metadata | Visual Pipeline (produces) | Playbook scripts (consume via --visual-identity) | LIVE | Schema in tools/visual/pipeline.py |
| Business Intelligence | Aggregate niche performance data across users and playbooks | API /v1/metrics, /v1/playbook (collect execution data) | Playbook authors (better niche selection), Karpathy Loop (optimize business outcomes) | DESIGNED | api/ (partial) |
| Instruction Reliability | Aggregate step compliance data per model, per instruction pattern | API /v1/insights, /v1/score (collect reliability data) | Prover (smarter fixes based on model-specific patterns), Authors (know which patterns work per model) | DESIGNED | api/ (partial) |
| API: /v1/score | Submit and retrieve quality scores | LIVE | api/src/routes/ |
| API: /v1/benchmarks | Benchmark data for calibration | LIVE | api/src/routes/ |
| API: /v1/metrics | Execution metrics across playbooks | LIVE | api/src/routes/ |
| API: /v1/playbook | Playbook metadata and registration | LIVE | api/src/routes/ |
| API: /v1/insights | Proven Laws and fix data (historically "design-rules") | LIVE | api/src/routes/ |
| API: /v1/pri | Playbook Reliability Index scores | LIVE | api/src/routes/ |
| Scaffold (new-playbook.py) | Generates complete playbook structure from template | LIVE | tools/scaffold/new-playbook.py |
| sync-public.sh | One-way sync: monorepo methodology + evidence to Proven-sh/proven public repo | LIVE | tools/publish/sync-public.sh |
| doc-health.sh | Documentation correctness + freshness audit (13 checks). Run before committing. | LIVE | tools/checks/doc-health.sh |
| cold-start-test.sh | Tests that CLAUDE.md bootstrapping instructions work. Launches claude --print, checks responses. | LIVE | tools/checks/cold-start-test.sh |
