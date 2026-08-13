# cosmo — FUTURE.md

Parked ideas and forward decisions. **Nothing here is in scope** until it's pulled into
SPEC.md via a version bump at a phase gate. Claude Code: read for context, do not build
from this file. Each item carries its earliest sensible phase.

Items tagged **[spec-1.5]** are near-term enough to fold into the spec at the next gate.

---

## Deployment tiers (target end-state)

1. **Static export** (`make render-static`) — read-only artifact for decks/email. P1.
2. **GitHub Pages** — CI-published `graph.json` + SPA bundle; read-only but interactive:
   client-side search, filters, saved views, deterministic queries (path, who-reads-X
   as ported/precomputed JS). Server surfaces (chat, writes, refresh) feature-flagged
   off when no server detected — one build, all tiers. Prereqs: CI-nightly collectors
   (forces ASK-DAN-5) and a deliberate visibility/governance decision — graph.json
   enumerates vuln posture per repo. (post-P3)
3. **Local served app** — full instrument (chat, writes, refresh). P4.

## New node/edge types

Each needs: collector, ID namespace, completeness field-set, renderer style (generic
styling covers unknowns), and a **join key** — the join key is always the real work.
Add demand-first, when a question needs them.

- `ticket:` Jira (P5, sketched in spec) — repo-tagging convention decided early.
- `job:` / `pipeline:` — Jenkins jobs and KFP pipelines as nodes ("which pipeline
  feeds this table").
- `directive:` — mandates as first-class nodes with edges to affected repos.
- `model:` — registered model distinct from its repo.
- **Teams, never individuals.** `team:` nodes with `owns` edges (sourced from curated
  `meta.owner`) answer "who do I talk to" at org-chart resolution. Individual person
  nodes — especially with activity data — flip the tool into people analytics and
  violate the binding fleet-not-people rule. Individuals are at most contact info on a
  team node, never nodes with metrics.

## Graph capabilities

- **Propagated risk scoring** — centrality-weighted: own vulns × downstream dependents
  along `reads` edges. A defensible ranking of where remediation hours go. (post-P3;
  pull-forward candidate #1)
- **Campaign sequencing** — topological sort → dependency-ordered rollout waves; the
  orchestrator consumes this. (with orchestrator integration)
- **What-if overlay** — "simulate deprecating table X" as a view op shading the
  affected subgraph. (post-P3)
- **Change correlation** — snapshot diffs joined against build failures: "these builds
  broke the day that table changed." (post-P2, data already exists)
- **Graph-RAG for DS support** — the graph as knowledge base behind the DS-deflection
  skill; lineage-path answers to "where does feature X come from." Converges two
  projects. (P4+)
- **Intra-repo semantic zoom** — a focused repo unfolds into its internal module/file
  subgraph (fleet dims around it). Per-repo graphs in `graph/repos/<id>.json`,
  lazy-loaded via an `expand` op; IDs extend as `repo:x/mod:y`. Module/file granularity
  only — never functions. (P5)

## The directive flow (the convergence demo)

Directive lands → "what repos are in scope?" (graph query over edges + scanned attrs)
→ subgraph → **+ save view** ("DIR-XXXX scope") → **⚑ campaign** on that selection →
orchestrator sequences waves, writes status back → nodes flip green on the scoping
view → evidence pack generated from the chain. Scope predicates the scanner doesn't
cover yet = **scan-on-demand**: Claude authors a parameterized scan rule (regex/AST
pattern) that executes deterministically across the mirrors; the approval gate shows
the **exact pattern**; proven rules get promoted into codescan's permanent set.

## Chat evolution (P4+ — the current design is deliberately ~60% of ceiling)

- **Investigations** — multi-turn agent loops where Claude calls `query_graph` /
  `run_scan` tools itself rather than one-shot answering from a JSON dump (the Agent
  SDK decision in disguise).
- **Conversational curation** — working the completion queue as an interview: drafted
  metadata offered approve/edit/skip. The mode that actually drains documentation debt.
- **Causal explanation** — chains, not summaries: "red because PR #77's kubekit bump
  introduced the pyarrow conflict."
- **Actions** — campaign kickoff from chat (see directive flow).
- **Proactivity beyond the briefing** — sparingly, or it becomes a nag.

## Charts / dashboard studio (post-P2 data, P3+ UI)

- Differentiator: **charts are scoped by the graph** — current selection / saved view
  is the chart's scope. `{"op":"chart","metric":"vulns.critical","scope":"selection",
  "range":"90d"}`.
- Arrives three ways for free: inline in chat, pinned to boards (a board = saved views
  generalized to a named list of chart specs), embedded in the weekly digest.
- v1 = a **trends** tab: criticals over time, compliance climb, staleness distribution,
  completeness progress.
- Guard: fleet metrics from snapshots only — do not rebuild Grafana.

## UX additions

- **[spec-1.5] Edge selection as first-class (R26 candidate)** — edges click to select:
  selection card shows kind/provenance/confidence/notes; "what's the story here?"
  resolves against the selected edge. (P3)
- **[spec-1.5] Export op** — `{"op":"export","format":"csv","scope":"selection",
  "columns":[...]}`; pure client-side blob download, works on every tier including
  Pages; column picker; todos table gets it free. Skip the Sheets API — CSV imports
  cleanly. (P3)
- **Second theme: quiet light.** Off-white ground, ink-gray edges, flat dots, zero
  glow, generous whitespace — engineering-document register, no cartography gimmicks.
  **Semantics are theme-invariant**: identical health hues (recalibrated for light
  ground), glyphs, rings. Light is the default for anything shared (static export
  always, Pages, projectors); nebula is the personal/wall-screen theme.
  **[spec-1.5]** Route all rendering through design tokens from the first SPA commit;
  glow is a theme property, not a constant. (tokens P3, second theme later)

## Virality / distribution (output travels, not the tool)

- **README health badges** — nightly-generated SVG per repo, committed by a campaign.
  Every repo README advertises cosmo. (post-P2; pull-forward candidate #2)
- **PR comments on edge changes** — "this PR adds a read of txn_agg — 3 downstream
  models." Useful at the moment of change, inside the surface engineers live in.
- **Shareable view snapshots** — current view state serialized into a self-contained
  mini-export for Slack. Permalinks once a shared (Pages/CI) instance exists.
- **Fleet replay** — snapshot history as a ~20s animation; quarterly-review opener and
  promo artifact.
- **The kit play** — generic naming already done; endgame is another team pointing
  cosmo at their org via config. Internal-open-source adoption story.

## Operational decisions

- **[spec-1.5] Scheduling:** launchd LaunchAgent (`StartCalendarInterval`) instead of
  cron — runs missed jobs after wake. Plus a staleness self-heal: collector runner and
  server check latest snapshot age on start and catch up if stale. Real fix remains
  CI-nightly (ASK-DAN-5). tmux is for long interactive campaign sessions, not
  scheduling.
- **[spec-1.5] Local mirrors for codescan:** bare/shallow clones under
  `~/.cosmo/mirrors/`, nightly fetch, scan working tree at pinned SHA. API for
  metadata; mirrors for code. Makes per-SHA caching natural. (P3)
- **[spec-1.5] Dev/prod separation:** `main` is prod (what launchd/CI runs and what
  Pages publishes); short-lived feature branches; phase gates become tags — no
  long-lived dev branch. The real risk is data, not code: `COSMO_DATA_DIR` env var
  (prod `./data`; dev `./data-dev`, gitignored). `make dev` seeds scratch by copying
  prod; data flows prod → dev, never backward — no promote command, by design. Dev
  collector runs hit scratch or replay recorded fixtures (`--fixtures` mode, extending
  the golden-file pattern); the dev server points at scratch so a write-op bug
  clobbers a copy.
- **[spec-1.5] Validation as commit gate:** the graph build schema-validates before
  any snapshot commits; on failure nothing lands, `_status.json` says why, and the map
  shows unknown instead of garbage — corruption is loud and cheap to undo, never
  silent. Annotations are the one unregenerable tree (human-authored): every write is
  already one commit (git history is the undo), and write-path changes get fixture
  tests before touching real data.
- **[spec-1.5] Snapshot lifecycle:** dirs are day-granular. Today's dir is a mutable
  working set — multiple runs (nightly, scoped `--only` refreshes, manual) update it
  in place, every run commits, so git holds the intraday states. Prior days are
  immutable (audit record + trend substrate). Trend extraction samples one state per
  day — the day's final — so refresh-heavy days don't read as denser signal in trend
  charts. Per-signal `as_of` carries the intraday truth regardless.
- **graphify as adapter, not replacement:** codescan (fleet data-IO edges) stays the
  core build — graphify does code-structure, not IO semantics, and cannot replace it.
  For intra-repo zoom, `collectors/graphify_adapter.py` transforms graphify output into
  cosmo's node/edge JSON (`source: graphify, confidence: 1.0`, kinds mapped to open
  strings). Checks when the time comes: Artifactory availability, granularity trimmed
  to module/file, version pinned (repin = re-verify, same rule as the SDK). (P5)

## Audience rollout sequence

Teammates: query skill, PR comments, badges ("not having to ask Dan").
DSs: lineage answers via graph-RAG. Manager: weekly digest, queues, trends. Leadership:
static exports, fleet replay, evidence packs, the propagated-risk slide.
**Sequence:** prove on self through P2 → teammates at the P3 gate → manager passively
via the digest → leadership only after weeks of accuracy track record. Trust bar rises
with each tier; one confidently-wrong red node in front of a skip-level costs more than
fifty right ones earn. The fleet-not-people rule gets more load-bearing at every step.

## Pull-forward priorities (first two, once P2 gates)

1. README health badges — small, loud, distribution built in.
2. Propagated risk scoring — small, and it's the slide leadership has never seen.
