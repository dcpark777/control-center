# cosmo — platform fleet map

**Spec version:** 1.4 · 2026-08-12 · (1.4: synced to mockup v6 — sidebar/briefing/saved-views/todos-view/multi-select folded in as R18–R25; reference mockup updated; "platform fleet" naming. 1.3: renamed atlas→**cosmo**; binding ID + schema-evolution rules. 1.2: served SPA primary + static export; P0 npm/vite check; scoped `--only` refresh + `/refresh` endpoint. 1.1: config.yaml tuning surface; SDK in-process tool check; ASK-DAN-4; P4 bridge spike)
**Owner:** Dan
**Executor:** Claude Code, following this document

A nightly-built, queryable graph of the platform fleet — every repo, model, and
data dependency as nodes and edges, overlaid with health signals — rendered as a
self-contained HTML file and (eventually) queryable from any Claude Code session.

Reference mockup: `docs/mockup.html` (cosmo-mockup-v6) is the validated interaction
model — unified right sidebar, briefing, saved views, todos view, multi-select. It is a
*design artifact*, not a code seed — its canvas renderer and CDN dependencies are
explicitly disposable. Treat it as the answer to "what should this look and feel like."

---

## How to execute this spec (read first, every session)

1. Read this file top to bottom once per session before writing code.
2. Read `PROGRESS.md`. If it doesn't exist, create it from the template in §9.
3. Work the **first unchecked task** in the current phase. Do not skip ahead across
   phases. Within a phase, tasks may be reordered if a dependency demands it — note why
   in PROGRESS.md.
4. Every task ends with: code committed, PROGRESS.md updated (checkbox + one-line note +
   date), and if the task has an acceptance check, evidence of it passing (command
   output pasted into PROGRESS.md).
5. **STOP gates:** each phase ends with a gate marked `⛔ GATE`. Stop there, summarize
   what shipped and what's uncertain, and wait for Dan's review before starting the
   next phase.
6. **ASK-DAN items:** anything marked `❓ASK-DAN` requires his input. Ask at the start
   of a session, don't silently guess.
7. Decisions not covered by this spec: prefer the *Design principles* (§2). If a real
   contradiction with the spec emerges, flag it in PROGRESS.md under "Spec issues"
   rather than silently deviating.

---

## 1. Context and constraints (non-negotiable)

- **Environment:** locked-down corporate sandbox. No external egress. Internal GitHub Enterprise,
  Jenkins, Mend/WhiteSource (vulns arrive as GitHub issues), Snowflake. Packages come
  from the internal Artifactory PyPI/npm mirrors only.
- **AI access:** `claude -p` (Claude Code non-interactive) is the only AI call path.
  No direct API keys, no custom MCP servers, no custom hooks (managed settings).
- **Language:** Python 3.11+, stdlib-first. Third-party Python deps require
  justification in PROGRESS.md; the default answer is no.
- **Frontend:** a locally served web app is the primary form (no desktop packaging —
  no dmg/Tauri/Electron, ever). A self-contained static export (`make render-static`)
  is retained as the shareable read-only artifact. All JS/CSS/fonts vendored from
  Artifactory mirrors (no CDN, no Google Fonts at runtime).
- **Data at rest:** JSON files committed to this repo. No databases in P1–P3.
- **Secrets:** never in this repo, never in snapshots, never in annotations. Tokens
  come from env vars; document required vars in `README.md`.
- **Politics:** the map describes *fleet health*, never people. No authorship
  leaderboards, no per-person metrics, no names on nodes.

## 2. Design principles (tie-breakers)

1. **Deterministic core, AI at the edges.** AI proposes; deterministic code validates
   and executes. AI output is always schema-validated and provenance-tagged.
2. **Trust over coverage.** A signal ships only when it's reliable. Unknown is a
   first-class, visible state — never guess green.
3. **Git is the audit layer.** Snapshots, annotations, approvals, and metadata edits
   land as commits.
4. **Thin and auditable over clever.** Prefer boring code someone else can read.
5. **Each phase is independently demo-able.** If a phase can't be shown, it's scoped
   wrong.

## 3. Repo layout

```
cosmo/
  SPEC.md              # this file
  PROGRESS.md          # living execution log (see §9)
  README.md            # setup, env vars, cron
  Makefile             # collect / build / render / serve / all
  collectors/          # one module per source, independently runnable
    config.yaml        # per-repo hints/overrides — the ONLY sanctioned tuning surface
    github_meta.py     # repo inventory, staleness, PR flow
    mend_vulns.py      # vuln counts by severity (from Mend GitHub issues)
    jenkins_builds.py  # last build status per job
    directives.py      # compliance flags (source per ❓ASK-DAN-1)
    issues.py          # open GitHub issues per repo   (P4)
    depscan.py         # requirements/conda dep lists  (P3)
    codescan.py        # IO edge extraction, 2-tier    (P3)
  snapshots/           # dated JSON, committed: 2026-08-12/github_meta.json ...
  graph/
    build.py           # merge snapshots -> graph.json
    schema.py          # node/edge/annotation validation (stdlib dataclasses)
    queries.py         # pure-python graph queries (P3)
  annotations/         # sidecar YAML per node/edge (P4 writes; readable earlier)
  render/
    grid.py            # -> static tile-grid page (P1; doubles as the static export)
    app/               # served SPA (P3): Vite + Preact + Sigma/WebGL, deps via
                       #   Artifactory npm; builds to dist/, served by server/
  server/              # local bus + write path + claude -p bridge (P4)
  skills/cosmo/        # Claude Code skill for query mode (P4)
  dist/                # build outputs (gitignored except tagged releases)
  tests/
```

## 4. Data model

### 4.1 IDs and node types

Namespaced, stable, lowercase: `repo:<name>`, `table:<schema>.<table>`, `lib:<name>`,
`pkg:<name>`, `ticket:<key>` (P5), `job:<jenkins-job>`. The `table:` namespace encodes
the roll-up hierarchy (schema-level grouping) — renderers may collapse on the segment
before the dot.

**IDs are permanent (binding).** When a source renames — e.g., a GitHub repo rename —
the node keeps its ID and the new source name maps to it via an alias entry in
`collectors/config.yaml`. Never mint a second node for a renamed thing: IDs leak into
layout.json, annotations, the audit log, and saved views, and a rename there is a
migration across all of them.

Node and edge **types are open strings**. Renderers must style unknown types generically
rather than erroring. Signals and metadata are open dicts — a new collector may add
keys without schema migration.

### 4.2 Node (graph.json)

```json
{
  "id": "repo:fraud-features",
  "type": "repo",
  "signals": {
    "vulns":      {"critical": 3, "high": 4, "source": "mend",    "as_of": "2026-08-12T03:10Z"},
    "build":      {"status": "fail",         "source": "jenkins", "as_of": "2026-08-12T03:12Z"},
    "staleness":  {"days": 2,                "source": "github",  "as_of": "2026-08-12T03:05Z"},
    "directive":  {"DIR-2026-03": false,     "source": "directives", "as_of": "2026-08-11T22:00Z"}
  },
  "meta": {
    "owner": null,
    "purpose": {"value": "...", "source": "ai", "confidence": 0.8, "verified": false},
    "links": {"repo": "https://...", "jenkins": "https://...", "logs": "https://..."}
  },
  "completeness": 0.45,
  "attrs": {"deps": {"snowflake-connector-python": "2.0.4"}}
}
```

Rules:
- Every signal carries `source` and `as_of`. A signal a collector failed to produce is
  **absent or `{"status":"unknown", "error":"..."}`** — never a stale value passed off
  as current, never a default.
- `meta` values are either plain (human-entered/verified) or provenance-wrapped
  (`source: ai`, confidence, `verified: false`). Renderers must distinguish them.
- `completeness` = weighted fill ratio of expected meta fields per node type. Expected
  fields for `repo`: owner, purpose, runbook, deploy, consumers. Defined in `schema.py`,
  adjustable.
- All URLs live under `meta.links.*`; renderers turn every entry into an action button.
- **Schema evolution (binding after the P1 gate):** signal and meta keys are never
  renamed or removed — deprecate in place and add the new key. Months of committed
  snapshot history and the trend extraction depend on key stability; additive change
  is free forever, destructive change never is.

### 4.3 Edge

```json
{"from": "repo:card-decision-model", "to": "table:features.features_v3",
 "kind": "reads", "source": "codescan-ast", "confidence": 1.0}
```

- `kind`: open string; initial vocabulary `reads | writes | depends | uses | tracks`.
- `source`: `codescan-ast` (deterministic) or `codescan-ai`; `confidence` mandatory for
  ai edges. Renderers draw ai edges visibly distinct (dotted/violet in the mockup).
- Packages are **attributes by default** (`attrs.deps`), promoted to `pkg:` nodes only
  by an explicit query/op. Never emit all dep edges into graph.json.

### 4.4 Annotations (sidecar, P4)

`annotations/repo:fraud-features.yaml` — list of items:

```yaml
- kind: todo            # note | todo
  text: Rotate Snowflake OAuth secret
  status: open          # open | done
  created: 2026-08-06
  done_at: null
  due: 2026-08-06
  recur: 90d            # optional; re-arms on completion
- kind: note
  target_edge: "table:features.txn_agg -> repo:card-decision-model"
  text: migrating to txn_agg_v2 in Q4
```

Completed recurring todos re-arm with a new `due`; the *completed entry is retained*
(it is the record, e.g. of a rotation). Overdue todos feed a `chores` signal at graph
build. Annotations never contain secret values — the chore, not the credential.

### 4.5 View-op protocol (P3 render, P4 live)

Ops are the only way any agent or UI changes the view. Renderer validates every op;
unknown node IDs or malformed ops are rejected and logged, never partially applied.

```
{"op":"focus","node":"repo:x","depth":1}
{"op":"filter","where":"<expr over signals/attrs>"}
{"op":"path","from":"<id>","to":"<id>"}
{"op":"materialize","package":"<name><spec>"}
{"op":"pin","node":"<id>"} / {"op":"reset"}
{"op":"annotate","node":"<id>","text":"..."}          # transient label
{"op":"set_meta","node":"<id>","field":"owner","value":"..."}   # write-tier
{"op":"add_todo","node":"<id>","text":"...","due":"...","recur":"..."}  # write-tier
{"op":"fetch","collector":"issues","nodes":[...]}     # external-tier
```

**Op tiers (policy lives in the server, never in the model):**

| tier      | ops                                   | policy                          |
|-----------|---------------------------------------|---------------------------------|
| view      | focus, filter, path, materialize, pin, reset, annotate | auto-apply     |
| write     | set_meta, add_todo, complete_todo      | approval chip; each = 1 git commit |
| external  | fetch                                  | approval chip; "always this session" scope, revocable, never persisted |

Every approved write/external op appends a line to `annotations/_audit.log`
(op, args, approver, timestamp) and is committed.

### 4.6 Query contract (P4)

`claude -p` is invoked with: `graph.json` + **current view state** (active selection,
last result set, pinned nodes) + conversation transcript, returning
`{"answer": "...", "view_ops": [...], "suggested_ops": [...]}` via `--json-schema`.
`view_ops` are auto-tier only; anything write/external must go in `suggested_ops` and
survive the server's tier check regardless. Default model: haiku; escalate to sonnet
when the question requires analysis (heuristic: server flag, not model choice).

## 5. Requirements from the v3 usability/trust audit

These are requirements, not suggestions. Phase mapping in parentheses.

- R1 search / jump-to-node, keyboard driven (P3)
- R2 durable answer transcript (P4)
- R3 hover tooltips: name + health line (P3)
- R4 op history with undo; reset is not the only recovery (P3)
- R5 single op pipeline; one visible current-state indicator; controls never disagree (P3)
- R6 label LOD by zoom/focus/hover (P3)
- R7 **pinned, deterministic layout persisted across snapshots** — positions stored in
  `graph/layout.json`, committed; new nodes placed without moving old ones (P3)
- R8 fit-all / fit-selection; minimap at >100 nodes (P3)
- R9 redundant non-color health channel (glyphs), grid mirrors it (P1 grid, P3 graph)
- R10 keyboard/screen-reader path: grid view is the accessible view; aria-live state
  announcements (P1 grid, P3)
- R11 first-run onboarding overlay, dismiss persisted per user (P3)
- R12 completeness legible at rest (hollow-core treatment for <50%) (P3)
- R13 ai-inferred edges visibly distinct + per-edge confidence in panel (P3)
- R14 per-signal freshness display; failed collector = banner + degraded header state (P1 data, P2 render)
- R15 session grants visible and revocable (P4)
- R16 unknown as first-class visual state; partial collector failure degrades visibly (P1 data, P2 render)
- R17 focused content fits the region left of the sidebar; sidebar width measured, not
  hardcoded (P3)
- R18 unified right sidebar: docked selection card + durable transcript + ask input;
  collapsible (⌘/); back mirrored in the sidebar header — one op history, two doors
  (P3 shell, P4 chat)
- R19 briefing card seeds the transcript from the snapshot diff — clickable lines,
  dismissible, reopenable from the sidebar header (P2 data, P3 UI)
- R20 ⇧-click multi-select; selection card shows the set; current selection is the
  query context ("these") (P3)
- R21 todos are a first-class view: overdue-first table, clickable node/edge links,
  recurrence badges, completed history retained (P3 read, P4 write)
- R22 saved views: any filter saveable as a chip; rename and delete supported;
  persisted per-user as serialized ops (P3)
- R23 edge hover: highlight + provenance tooltip (kind · source · confidence) (P3)
- R24 selection card opens with a one-line verdict — worst signal first + documented %;
  keyboard zoom/pan (+/−/0/arrows) (P3)
- R25 product language is "platform fleet" — generic, never org- or domain-specific (P1)

## 6. Phases

### P0 — Pre-flight (one session)

Goal: know what the sandbox actually permits before writing real code. Record every
result (command + outcome) in PROGRESS.md under "P0 findings".

- [ ] **Artifactory/npm check:** are `sigma` (or an equivalent WebGL graph lib),
      `vite`, and `preact` available from the internal npm mirror, and does a
      hello-world `vite build` succeed offline? If no WebGL lib: P3 falls back to
      canvas with LOD (mockup-proven to ~100 nodes). If the vite toolchain itself is
      unavailable: P3 falls back to a served vanilla-JS app — flag at the P0 gate.
- [ ] **GitHub Enterprise API:** with Dan's token from env, list repos in the fraud org,
      fetch last-commit date for one repo, list open issues for one repo. Confirm
      pagination and rate limits.
- [ ] **Mend signal:** confirm vuln data is reachable as GitHub issues (labels? title
      convention?) and parse severity for one repo. Document the parse rule.
- [ ] **Jenkins API:** fetch last-build status for one known job. Document auth method.
      Also capture the job-naming convention → `❓ASK-DAN-2` if repo→job mapping isn't
      derivable.
- [ ] **`claude -p --json-schema` smoke test:** trivial prompt → validated JSON out.
      Record model names available.
- [ ] **cron/scheduling:** confirm how a nightly job runs in this environment (cron?
      launchd? CI job?). Document the chosen mechanism.
- [ ] **Agent SDK in-process tool check:** extend the existing `nova_feasibility.py`
      (July run passed on the work machine: SDK→Bedrock, turns/edits/hooks/resume)
      with one new check — define a trivial `@tool` via the SDK's in-process MCP
      server, ask a question only that tool can answer, assert the tool fired. Pin the
      SDK version that passes.
- [ ] ❓**ASK-DAN-4:** policy interpretation — does "no custom MCP servers" cover an
      in-process SDK tool inside our own program? Get the platform admin's answer and
      record it verbatim in PROGRESS.md.
- [ ] ❓**ASK-DAN-1:** directive/compliance data — parseable source of truth, or do we
      start with a hand-maintained `snapshots/manual/directives.yaml`?
- [ ] ❓**ASK-DAN-2:** confirm repo list scope for v0 (the ~40 fraud repos — from a
      GitHub org/team query, or an explicit list file?)
- [x] ✅**ASK-DAN-3 (resolved 1.3):** tool name is **cosmo** — the menubar app of the
      same name is retired and the name transferred. Repo `cosmo`, config `~/.cosmo/`
      free to reuse.

`⛔ GATE P0` — summarize findings; any collector that failed reachability gets its P1
task rewritten as file-drop import before proceeding.

### P1 — Walking skeleton (v0: replaces the hand-maintained tracker)

Goal: nightly snapshots + a tile-grid HTML Dan can attach to an email.

- [ ] `schema.py`: node dataclasses + validation; signals with source/as_of; unknown
      state representation. Unit tests.
- [ ] `collectors/github_meta.py` → `snapshots/<date>/github_meta.json`
- [ ] `collectors/mend_vulns.py` → severity counts per repo
- [ ] `collectors/directives.py` → per ASK-DAN-1 outcome (manual YAML acceptable)
- [ ] Collector contract: each is independently runnable, exits nonzero on failure,
      writes `_status.json` (ok/failed/partial + error) alongside its output, and
      accepts `--only <node-ids>` / `--source` scoping for targeted refresh. A failed
      collector never blocks the others (R14/R16 start here, in the data).
- [ ] Tuning surface: collectors read per-repo hints from `collectors/config.yaml`
      (nonstandard requirements paths, Jenkins job-name overrides, table aliases). No
      hardcoded per-repo special cases in collector code — hints live in reviewable,
      diffable config.
- [ ] `graph/build.py`: merge latest snapshot set → `graph.json` (nodes only, no edges).
      Missing collector output ⇒ affected signals marked unknown.
- [ ] `render/grid.py` → `dist/cosmo.html`: self-contained tile grid, worst-first sort,
      health colors + glyphs (R9), completeness bar, per-signal freshness stamps,
      degraded-collector banner (R14/R16). Vendored font or system stack. Tiles are
      buttons with aria labels (R10).
- [ ] `Makefile`: `make collect build render` and `make all`; README with env vars +
      nightly schedule per P0 finding.
- [ ] Commit a real snapshot; verify `git log` diffing two snapshot days works as the
      trend substrate.

Acceptance: `make all` from a clean clone (with tokens) produces `dist/cosmo.html`
showing real fleet data; killing one collector mid-run yields a rendered page with a
visible degraded banner and unknown states, not a crash and not fake green.

`⛔ GATE P1` — demo to Dan. Decide: does v0 already replace the tracker? What's wrong
with the tile design against the mockup?

### P2 — Leadership artifact (v1)

- [ ] `collectors/jenkins_builds.py` (per P0 findings)
- [ ] Trend extraction: walk snapshot history in git → `trends.json` (critical vulns,
      stale count, compliance % over time); sparkline block on the grid page
- [ ] Weekly digest: `make digest` → markdown summary of snapshot-over-snapshot diff
      (deterministic diff; one optional `claude -p` pass for prose), suitable for Slack
- [ ] Anomaly lines in the digest (new criticals, newly stale, compliance regressions)
- [ ] Curation queue v0: rank = importance(degree×activity, degree=0 pre-edges → use
      activity) × incompleteness; render as a section on the grid page

Acceptance: digest for a real week reads correctly; trend numbers reconcile with
manual snapshot diff.

`⛔ GATE P2`

### P3 — The map (v2: edges + graph view)

- [ ] `collectors/depscan.py`: requirements/conda/pyproject parse → `attrs.deps`
- [ ] `collectors/codescan.py` tier 1: AST + heuristics for Snowflake/S3 IO across
      heterogeneous repos (no shared IO layer exists — assume nothing); emits edges
      with `source: codescan-ast, confidence: 1.0`; caches per commit SHA
- [ ] `codescan.py` tier 2: ambiguous files only → `claude -p --json-schema` →
      `source: codescan-ai` + confidence; hard cap on files per repo per run
- [ ] `graph/build.py`: edges + `table:` roll-up metadata; `graph/queries.py`
      (neighbors, path, blast-radius, attr filters) with tests
- [ ] Graph view as served SPA (`render/app/`: Vite+Preact+Sigma per P0 findings,
      else served vanilla+canvas): nebula aesthetic from mockup v6, collapse-first
      schema roll-up, R1 R3–R8, R11–R13, R17. `make render-static` (grid page)
      retained as the read-only export.
- [ ] App shell per mockup v6 (R18–R25): right sidebar with selection card (transcript
      placeholder until P4), todos view reading `annotations/`, saved views as
      serialized ops in a per-user file, briefing card from P2 trends, multi-select,
      edge hover, verdict line, keyboard nav.
- [ ] `graph/layout.json`: persisted positions; incremental placement for new nodes (R7)
- [ ] Provenance rendering + per-edge confidence in panel (R13)

Acceptance: "which repos read `table:X`" answered by `queries.py` matches a manual
check on 3 repos; layout identical across two consecutive builds; one deliberately
mis-parsed repo shows as ai-inferred edge, visually distinct.

`⛔ GATE P3` — this is the "which repos read from Snowflake" demo.

### P4 — Live instrument (v3: talk to the map)

- [ ] **Opening spike:** decide the query-bridge substrate — `claude -p` vs Agent SDK —
      from the P0 in-process tool result + the ASK-DAN-4 answer. Record decision and
      rationale in PROGRESS.md. The bridge module isolates the choice either way.
- [ ] `server/`: local http server (stdlib vs FastAPI per opening spike); serves dist +
      graph.json; view-state endpoint (poll or SSE); op validation + **tier policy
      table** (§4.5); audit log; `POST /refresh?collector=X&nodes=...` scoped refresh
      (human-initiated — no approval gate; the `fetch` op remains the gated in-chat
      form). On-demand results update the working set; nightly commits remain the
      canonical dated snapshots. UI: ↻ on node panel + retry on the degraded banner.
- [ ] Write path: set_meta/add_todo → annotations YAML → git commit per change;
      approval chips; session grants visible + revocable (R15)
- [ ] `collectors/issues.py` + `fetch` op (external tier) with the paused-loop resume
- [ ] Query bridge: §4.6 contract, selection-as-context, suggested-ops-as-chips,
      durable transcript (R2); haiku default
- [ ] `skills/cosmo/`: second-screen mode — Claude Code skill that loads graph.json,
      answers in terminal, writes view-ops to the bus (CC native permissions = the gate)
- [ ] Embedded chat sidebar (mode 2), reusing the same protocol

Acceptance: the two-step flow from the mockup runs end-to-end on real data — package
filter → "do these repos have open issues mentioning X" → approval → fetched issues —
with the audit log showing the approved fetch.

`⛔ GATE P4`

### P5 — Later (not yet specced in detail; do not start)

Jira collector + repo↔ticket convention; orchestrator hooks (map emits target lists,
campaigns write status back); saved views / guided tours; evidence-pack generation;
override→repo-PR promotion campaign.

## 7. Testing & quality bar

- Unit tests for schema validation, graph build merge rules (esp. unknown handling),
  queries, and op-tier policy. `make test` green before any gate.
- Collectors: golden-file tests with recorded API responses (scrubbed).
- No test framework beyond `pytest` (justify in PROGRESS.md if even that is unavailable;
  stdlib `unittest` acceptable).
- Every AI-touching path (codescan tier 2, query bridge) has a test with a *canned*
  claude response — tests never call `claude -p`.

## 8. Security & data hygiene

- Tokens via env only; `_status.json` and logs must never echo them.
- Snapshots contain no secrets, no personal data, no commit-author analytics.
- `set_meta`/todo text is free text — the UI reminds "the chore, not the secret" on
  credential-related keywords (simple wordlist nudge, not a blocker).
- The repo is assumed readable by the team from day one; write nothing you wouldn't
  show your skip-level.

## 9. PROGRESS.md template & protocol

```markdown
# cosmo — progress log
Current phase: P0
Last session: <date> · <one-line summary>

## P0 findings
| check | result | evidence / notes |
|-------|--------|------------------|

## Task log
- [ ] P0: artifactory/npm check
  ...(mirror every checkbox from SPEC §6; add date + note when checked)

## ASK-DAN queue
- ASK-DAN-1 (directives source): OPEN

## Spec issues
(none)

## Decisions made in-flight
(date · decision · why)
```

Protocol: update PROGRESS.md **in the same commit** as the work it describes. Session
start = read SPEC + PROGRESS; session end = "Last session" line updated. If a session
is interrupted, PROGRESS.md must still be accurate — it is the only memory between
sessions.

## 10. Considered alternatives (decided — do not relitigate without the named trigger)

- **Renderer — single-file-first:** superseded in 1.2. The email/portability
  requirement was dropped, so the served SPA is primary; the static grid export
  preserves the deck/leadership artifact. Desktop packaging (dmg/Tauri/Electron)
  rejected permanently.
- **Storage — SQLite:** rejected. Loses the free diffable audit trail and trend
  substrate that git-committed JSON provides. **DuckDB:** complementary, not a
  replacement — it can query `snapshots/**/*.json` in place for trend analytics.
  *Trigger:* P2 trend extraction exceeds ~150 lines of hand-rolled git-walking, or
  ad-hoc analytics questions arrive.
- **Graph — networkx:** deferred, pre-approved. *Trigger:* the curation-queue
  importance score needs real centrality measures. **Graph databases (Neo4j, embedded
  Cypher stores):** rejected at this scale — server/approval/ops weight for queries
  that fit in memory.
- **Server — FastAPI:** genuinely open. Decide in the P4 opening spike alongside the
  bridge substrate; op validation is a security boundary and may justify Pydantic.
  stdlib `http.server` remains the default and P1–P3 need no server at all.
- **Scanner — tree-sitter:** deferred. *Trigger:* non-Python sources (SQL files,
  Scala) need edge extraction. **Semgrep:** rejected — approval weight exceeds value
  over tier-1 heuristics + tier-2 Claude.
- **Scheduling — CI-nightly (Jenkins job runs the collectors):** deferred; laptop
  cron for P1. *Trigger:* anyone besides Dan depends on the artifact's freshness —
  then raise as ASK-DAN-5 (credentials-in-CI conversation).
- **AI path — direct Bedrock via boto3:** escape hatch only. Bypasses the managed
  Claude Code guardrails that keep this legible to the org; do not explore unless
  ASK-DAN-4 forbids the SDK *and* `claude -p` proves insufficient.

