# START_HERE — build instructions for Claude Code

You are picking up **mission**: a mission-control framework for Claude Code.
The design phase is COMPLETE. Your job is implementation, faithful to the
documents in this repo. The human you are working with is **Command** (Dan);
defer every judgment call the spec doesn't settle to them.

## Read order (do this first, fully, before writing anything)

1. `SPEC.md` — the architecture and mechanics. It is authoritative.
2. `mission-skill/SKILL.md` — the Captain's judgment codex (§11 failure
   table especially).
3. `mission-runtime/harness.md` — the protocol wrapper contract.
4. `mission-runtime/crew.toml` — every tunable knob; nothing numeric is
   hardcoded anywhere.
5. `mission-runtime/evals/mission-evals.json` — the acceptance suite the
   Captain must eventually pass.

## Non-negotiable build rules

- **Deterministic core, AI at the edges.** The `mission` CLI is plain
  Python: parsing, state machine, events, checkpoints, rendering. No AI
  calls inside the CLI. No AI behavior in code, ever — all judgment lives
  in the markdown files.
- **One truth.** `mission.json` + `events.log` are the only runtime state.
  Every renderer reads them; nothing else holds state.
- **Harness and knobs load from disk** (`harness.md`, `crew.toml`) — never
  inline them. Tunability is a core requirement.
- **Spec wins.** Where the spec and your instinct differ, the spec wins.
  Where the spec is silent or ambiguous, STOP and ask Command — do not
  fill gaps with design of your own.
- **No v2 features.** Sequential execution only, board.html read-only,
  no daemon, no watchers, no parallel worktrees. The v1 cutline in
  SPEC.md §13 is a fence.
- Test-first for the CLI. Small commits with clear messages; git history
  is the audit trail.

## Phase plan (one phase per session is fine; finish a phase before starting the next)

**Phase 0 — environment verification (~15 min).**
Confirm and report to Command: Claude Code version (skills/commands
unification present); subagent dispatch works in this environment; the
`statusLine` settings command is permitted (SPEC open question 2 — test
it, record the answer in SPEC.md §14). Then `git init`, commit the bundle
as-is: `baseline: design artifacts v0`.

**Phase 1 — toy fixture repo.**
Build `fixtures/toyrepo/`: a small Python project deliberately broken to
match the eval setups — a failing retry test with a changed default
timeout, a parallel-unsafe shared fixture behind flaky auth tests, a
misspelled gate path that makes pytest collect 0 tests. Acceptance: each
eval case's `setup` line has a matching reproducible state.

**Phase 2 — the `mission` CLI skeleton.**
Implement per SPEC §5–6: `compile` (dialect → plan.json, refusing
ambiguity with location), `status`/`done` (state mutations + event
append), checkpoint commits, transcript-board render, `why`, `init`,
bare-`mission` where-was-I, `resume`, `abort`, `revert`, `crew lint`.
Acceptance: unit tests green; compiling the SPEC §6.1 example plan
round-trips correctly; a hand-driven fake mission (CLI calls only, no AI)
walks the full state machine cleanly.

**Phase 3 — wire the Captain.**
Install `mission-skill/` at `.claude/skills/mission/`, members via
`mission crew sync`. Run one real end-to-end mission on the toy repo
(the failing retry test — eval case `size-small-zero-questions`).
Acceptance: fast path fires, gate runs, checkpoint exists, board renders,
wrap reports honestly.

**Phase 4 — eval pass.**
Run the full eval suite as scripted scenarios; grade per case; record
results in `evals/results/`. Failures produce codex/harness/knob tuning
proposals for Command — never silent codex edits. Acceptance: calibration
cases pass; every failure has a written tuning proposal.

STOP after Phase 4. Real-repo dogfooding is Command's call, not yours.

## When you finish any phase

Summarize: what was built, what deviated from spec (should be nothing —
explain if not), what's ambiguous, what Command should decide next.
Keep it short and concrete.
