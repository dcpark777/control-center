# cosmo — AMENDMENTS.md (staging for spec 1.6)

Decisions made in design sessions after spec 1.5 / the P1 gate. This file is a
**staging area, not documentation**: nothing here is buildable scope until
consolidated into the proper doc. Claude: read for context, do not build from
this file (same rule as FUTURE.md). Consolidation protocol at the bottom.

Tags say where each item lands:
[SPEC-RULE] spec body (§1–§4) · [SPEC-P3]/[SPEC-P4] phase task text ·
[SPEC-§10] considered-alternatives · [FUTURE+]/[FUTURE−] add/remove in FUTURE.md ·
[PB] PLAYBOOKS.md edit · [CLAUDE] CLAUDE.md edit · [ASK-DAN] open question ·
[DAN] human task, not Claude's

---

## Scanners & rule engine

1. [SPEC-P3] Codescan tier 1 is **sink-anchored**: table edges are emitted ONLY
   from string values reaching a configured IO sink (spark.read.table/saveAsTable,
   spark.sql, pd.read_sql, cursor.execute, snowflake.connector, …). No dotted-text
   harvesting — `os.path`-class false positives must be structurally impossible.
   Add the os.path repro as a regression fixture.
2. [SPEC-P3] **sqlglot** (Snowflake dialect) parses SQL-string sink arguments;
   table refs from its AST only; CTE aliases excluded.
3. [SPEC-P3][SPEC-§10] Rule engine: **Opengrep preferred, contingent** on
   (a) intake spike result and (b) golden-corpus bake-off vs the sink-anchored
   stdlib visitor (compare precision/recall AND unresolved-queue size — the
   constant-propagation payoff). ast-grep (pip) is the lighter fallback; ONE
   engine maximum; the stdlib sink-visitor remains the zero-dep floor either way.
   Rewrite §10's Semgrep rejection: Semgrep re-licensed Dec 2024 → Opengrep is
   the LGPL-2.1 community fork (10+ vendor consortium), rule-format compatible.
4. [SPEC-P3] Tier 2 is **retrieval-first**: input = specific unresolved sink
   sites from tier 1; per site, the enclosing function source (via codegraph CLI
   or AST slice), ONE narrow question, JSON-schema output, explicit `unknown`
   allowed. Self-consistency: 2–3 runs, accept on agreement else unknown.
5. [SPEC-RULE] Confidence is **computed, not assigned**: two independent
   deterministic detectors agree → 1.0; single deterministic → 0.9; tier-2 ≤ 0.8
   and never above any deterministic source. Precedence: deterministic wins on
   contradiction; contradictions logged to the audit sample.
6. [SPEC-P3][DAN] **Golden corpus**: ~5 hand-labeled repos (Dan labels);
   per-detector precision/recall computed in CI on every scanner/rule change.
7. [SPEC-P3] Weekly **audit sampling** of tier-2 edges (small random set for
   human verify) feeding the accuracy metric.

## Deps accuracy (depscan)

8. [SPEC-P3] Truth ladder: parse **lockfiles first** (resolved, incl.
   transitives; flags `direct`/`transitive`); manifests fallback. For lockless
   repos, derive resolution via **uv** against Artifactory. Provenance
   distinguishes `resolved(lock)` vs `resolved(cosmo-derived)` — derived gets
   lower confidence (may differ from the repo's actual deployed env).
9. [SPEC-P3] **deptry** compares imports vs declarations → signals
   `unused_deps` and `undeclared_imports` ("declares X" vs "actually imports X").
10. [SPEC-P3] **packaging** for PEP 440/508 parsing; query evaluation uses real
    SpecifierSet semantics. Schema attrs get types (version, enum, image-ref)
    with type-aware matchers.
11. [SPEC-P3] depscan scope adds: Dockerfile FROM → `attrs.base_image`;
    python version (Dockerfile/.python-version/requires-python) →
    `attrs.python_version`. Blessed-images list in config.yaml → derived signal
    `base_image_outdated` (attr from scanner + judgment from config pattern).

## Schema & trust rules

12. [SPEC-RULE] **Evidence pointers** required on every scanned edge:
    `{file, line, rule_id, content_hash}`. Shared artifacts (graph.json, Pages)
    carry pointers ONLY — snippet text renders on local tiers by reading the
    mirror. No source fragments leak to shared tiers.
13. [SPEC-RULE] **Table-identity normalization**: canonical lowercase,
    qualified against connection defaults where resolvable; one table = one
    node regardless of casing/qualification in source.
14. [SPEC-RULE] Scan artifacts stamped with engine versions + ruleset hash
    (same SHA + same versions ⇒ identical output, tested).
15. [SPEC-RULE] Per-repo `scan_coverage` attr (files parsed vs skipped) —
    absence of an edge is only meaningful relative to coverage.
16. [SPEC-RULE] Manual-source validation failures (e.g. bad category enum
    value) **degrade** (field → unknown + flagged), never halt the build.
    Halt remains reserved for schema-invalid merge output / corruption.
17. [SPEC-RULE] Never attach git-blame/author identity to evidence or any
    signal (fleet-not-people).

## Architecture

18. [SPEC-RULE] **Two-tier storage**: Tier A = per-repo artifact store
    (mirror + codegraph.db + scan.json + deps.json, SHA-keyed, derived,
    gitignored); Tier B = fleet index (graph.json: attrs, signals, edges).
    Query planner pushes deep predicates down to Tier A only for repos passing
    the Tier B filter. Deep queries are a LOCAL-tier capability — Pages/shared
    clones hold Tier B only; UI states this ("deep scan: local instance only").
19. [SPEC-P3][SPEC-P4] **cosmo-cli**: queries.py is built as the CLI core at P3
    (`cosmo query --json`, `cosmo node --json`, `cosmo status`); P4 adds
    `cosmo manifest --view X` and `cosmo campaign report`. Skill distribution =
    instructions + CLI on PATH. Executor v0 = shell loop over
    manifest → claude -p per repo → campaign report.
20. [SPEC-P4] **Selection manifest** contract: {query, as_of, node ids,
    relevant attrs, repo/mirror paths, match provenance} — the handoff artifact
    between cosmo (targeting) and any executor (action). Pull-forward candidate.
21. [SPEC-RULE] codegraph adopted for tier-2 retrieval + intra-repo zoom
    backend. Pinned version; SQLite schema treated as external contract
    (repin ⇒ re-verify). Prefer Artifactory npm over the curl installer;
    record installed version + binary hash meanwhile. [DAN]

## Metadata & context

22. [SPEC-RULE][DAN] `meta.category` enum {batch, kfp, realtime}, manual YAML
    per PB-1, enum-validated. Dan populates the initial 61 entries.
23. [SPEC-§1] Context edits: fleet is **61 repos** (not ~40); egress is
    **selective** (raw.githubusercontent.com reachable; git-to-github.com
    blocked) — treat reachability as incidental, nightly path assumes none.

## FUTURE.md edits

24. [FUTURE−] Zoom adapter: replace "graphify preferred" with codegraph
    (graphify demoted to alternative).
25. [FUTURE+] Mermaid export op (selection → GitHub-native diagram for
    PR comments/READMEs — pairs with badges).
26. [FUTURE+] scc repo-shape metrics + libyear-style dep-age (computable from
    attrs.deps + Artifactory metadata) as Fleet Risk Index inputs.
27. [FUTURE+] syft for image-level dep truth — trigger: when base-image
    analysis needs rigor beyond FROM-parsing.
28. [FUTURE+] Runtime lineage (OpenLineage/Spline listeners) as a future
    VERIFYING collector — observed edges confirming static ones.
29. [FUTURE+] cosmo.yaml in-repo descriptor: demoted to someday-option
    (rejected for now — no cosmo artifacts in other repos; metadata stays
    centralized/manual). Trigger: curation bottlenecks on Dan AND teams ask
    to contribute.
30. [FUTURE+] Notebook scanning (nbformat/jupytext) deferred — trigger: DS
    notebook IO becomes query-relevant.
31. [FUTURE+] Repo-expert agents + escalation ladder (ad-hoc rule → codegraph
    query → agent) as lane 2; promotion rule: a lane-2 question asked ~3×
    becomes a lane-1 detector.

## Governance doc edits

32. [CLAUDE] Replace "Python stdlib first" with: dependencies allowed when
    clearly beneficial; every new dependency still gets a one-line
    justification in PROGRESS.md and a pinned version.
33. [PB] PB-3: update engine story (Opengrep/ast-grep rules or stdlib visitor
    behind the registry; sqlglot for SQL args; sink-anchored requirement;
    fixtures-first unchanged). PB-1: note enum validation for
    allowed-value fields.

## Open items on Dan (not Claude's)

34. [ASK-DAN] 2b — Jenkins branch rule (blocks jenkins_builds.py, P2).
35. [ASK-DAN] 5 — CI-nightly (blocks Pages/sharing/badges; the bottleneck on
    everything social).
36. [ASK-DAN] 4 — SDK in-process tools policy (needed before P4 bridge spike).
37. [DAN] Pre-P3 **intake spike**, one sitting: Artifactory/appsec check for
    opengrep (or blessed semgrep), sqlglot, uv, deptry, packaging,
    ast-grep-cli, codegraph npm.
38. [DAN] Golden-corpus labeling (item 6) before codescan build starts.

## Campaign readiness (added after execution-path review)

39. [SPEC-P3] **Lens ops**: `color_by` as view-state dimension —
    `{"op":"lens","by":"health|type|category|compliance|staleness|completeness"}`.
    Health glyphs (✕/!/?) persist across every lens; legend switches with lens;
    lenses serialize into saved views. Colorblind-safe discrete palettes for
    categorical, ramps for continuous.
40. [SPEC-P4] **Campaign file contract**: `data/campaigns/<id>.yaml` — scope
    query, **manifest frozen at launch**, per-repo attempt states
    (pending/proposed/approved/merged/verified/failed), wave assignments.
    Nightly rescan changing who *would* match = scope drift, flagged separately,
    never silently added to a running campaign.
41. [SPEC-P4] Executor model is **propose-then-batch-approve**: org policy
    gates git push/PR creation on human approval, so the executor autonomously
    prepares local branches/diffs + a review queue; Dan approves pushes in
    batches. Full autonomy is impossible by policy — design for it, don't
    fight it.
42. [SPEC-RULE] **Campaignable = scannable**: a campaign may only target an
    attribute cosmo can verify by rescan (depscan attr, detector result).
    The scanner is the verification loop; no campaign without its verifier.
43. [SPEC-P4] Campaign conventions: one branch name across all repos
    (`cosmo/<campaign-id>`), commit/PR-body template carrying campaign id +
    directive link + evidence — the PR body is also the distribution surface.
    **Wave pacing is a social feature**: rate-limit PRs/day; a campaign is an
    act upon other teams, not just repos.

## Scanner power, curation, UI & tooling (later sessions)

44. [FUTURE+] **DEFERRED** — snowflake_catalog collector (manual-export mode:
    Dan drops a worksheet CSV export at data/manual/; whitelist validation of
    candidate table strings; ACCESS_HISTORY verification later). Moved to
    FUTURE.md with trigger: when codescan FP rate stays material after
    sink-anchoring + corpus hardening, or when a sanctioned query venue
    (KFP/CI) appears. Principle stands: export-and-drop is a first-class
    collector input mode.
45. [SPEC-P3] **Helper two-hop resolution**: detect in-repo helpers wrapping
    sinks (`def load(t): return spark.read.table(t)`) → treat helper as
    derived sink → resolve ITS call sites (codegraph callers query).
    The major recall unlock in org code, where IO is always wrapped.
46. [SPEC-P3] **Template ancestry**: map repos to their cookiecutter/template
    ancestors (fleet likely descends from ~3); scan templates themselves;
    per-template detector packs + priors. Pairs with the one-time
    fleet idiom census.
47. [SPEC-P3][SPEC-P4] **Correction flywheel**: `cosmo flag <fact>` CLI →
    auto-creates regression-fixture stub + audit entry; curation queue
    surfaces low-confidence × high-centrality facts for human confirm →
    `verified: true` provenance at confidence 1.0, confirmed facts become
    fixtures. Derived cross-collector signals (mend vuln × deptry
    actually-imported; staleness × build × dep-age) as the cheap
    "power" layer.
48. [SPEC-P3][SPEC-P4] **Curate surface** — Dan answers, cosmo learns.
    Feed (one ranked worklist): missing expected-meta, low-confidence facts
    to confirm, unresolved tier-2 sink sites (with file:line evidence from
    the local mirror + candidate answers incl. "unknown"), flagged facts,
    triangulation disagreements. Ranked by centrality × staleness ×
    campaign relevance. Every answer = a write with `verified` provenance +
    audit commit; confirmations auto-become fixtures (feeds #47).
    **v0 = `cosmo curate` CLI** (terminal Q&A writing manual/annotation
    files + committing — needs no server, pull-forward candidate, P3);
    v1 = fourth SPA view at P4 (keyboard-driven y/n/skip/type-ahead);
    the FUTURE.md "conversational curation interview" = same feed via chat.
49. [SPEC-P3] **UI review harness** (`make ui-review`) — CLI-only, no MCP:
    Playwright as a plain script using the SYSTEM Chrome
    (channel="chrome" — avoids browser-binary downloads/egress). Scripted
    tour: load → focus node → open panel → switch lens/view → per step,
    (a) screenshot PNG, (b) scene-state JSON dump (window.__cosmo_state:
    positions, colors, visible labels), (c) console log/errors → all to
    review/. Claude Code reads PNGs natively and iterates: edit → make
    ui-review → look → fix. State dumps are the correctness channel
    (assertable, goldenable); screenshots are the aesthetics channel;
    pixel-diffing avoided (fragile). Document the target in CLAUDE.md so
    sessions use it unprompted.

50. [SPEC-P3] Near-term adoptions (each at its task, pinned + one-line
    justification): **typer** (cosmo-cli subcommands as they multiply:
    query/node/status/curate/flag/manifest/campaign), **questionary**
    (curate-CLI prompts: y/n/skip, type-ahead), **jsonschema** (the
    validation gate's schemas as declarative, versionable files — the
    contract artifact skill-consumers can read), **deepdiff** (snapshot→
    snapshot change trees powering briefing/digest generation), **jinja2**
    (digest + PR-body templates), **pytest-regressions** (golden-file
    tests stop being hand-rolled). Considered-rejected: GitPython/pygit2
    (subprocess git stays — most auditable), pydantic (jsonschema lighter,
    schema-as-file wins), Textual (curate v1 option only).
51. [SPEC-P4] **Campaign edits prefer codemods over per-repo LLM edits**:
    for mechanical transforms (the jinja-env migration class), Claude
    authors ONE LibCST codemod, it's reviewed like a scan rule (gate shows
    the transform), then applied deterministically fleet-wide; per-repo
    LLM editing reserved for genuinely non-mechanical changes. One reviewed
    transform beats 61 independent edits — deterministic core applied to
    the write path. (LibCST: lossless Python CST + codemod framework.)
52. [FUTURE+] **vega-lite** as the chart-op engine when charts land —
    chart specs ARE JSON, matching the ops-are-data design (chart op emits
    a spec, renderer embeds it); also dagre/ELK for layered-DAG views
    (campaign waves visualization). rich (already a dep) to be exploited
    for CLI tables/progress; ripgrep for the idiom census; hypothesis
    property tests for table-identity normalization.

53. [SPEC-P3] **Renderer: custom canvas 2D, not Sigma.** At fleet scale
    (~100–300 nodes) WebGL buys nothing, and cosmo's visual grammar
    (hollow/filled, completeness rings, hatched unknown, glyph badges,
    lens palettes, provenance edge styles) is the product — every element
    is trivial in canvas and a custom node-program fight in Sigma. Port
    the mockup-v7 renderer (already implements the full grammar +
    pan/zoom/hover/click/⇧-multi-select/edge hit-test/minimap/LOD labels,
    validated over 7 iterations) into the SPA as a component reading
    layout.json + view state. Sigma/graphology demoted to §10 with
    trigger: node count approaches ~1–2k or measured frame-rate pain.
    Library fallback if hand-rolled interactions ever chafe: Cytoscape.js
    (canvas, rich styling), not Sigma. Dynamics rule: animate camera,
    emphasis (dim/highlight/pulse), and presence (fade in/out) via rAF
    tweens — geography never animates; layout stays pinned (no physics
    wiggle, by design). Position tweens only on explicit layout change.

54. [SPEC-P3] **Polish bar** (dedicated feel-pass task at end of P3; verified
    via harness + manual pass): zoom anchors to CURSOR, not canvas center
    (known mockup gap — fix in port); one MOTION token set (cubic ease-out,
    150–300ms, nothing >400ms) used everywhere; tweens interruptible
    (retarget from current, never snap); wheel-delta normalization
    (trackpad vs mouse); dim/undim and lens switches crossfade (~150ms),
    never pop; labels never overlap illegibly (LOD + halo); half-pixel
    alignment on 1px strokes; empty states designed (filter with zero
    matches → message + reset, never a blank void); dirty-flag rendering
    (skip redraw when nothing animates — silent fans on a work laptop are
    perceived quality); focus-visible rings + Esc-always-works; harness
    captures frame times during the scripted tour (p95 < 17ms budget).
    prefers-reduced-motion honored throughout (already in mockup).

55. [FUTURE+] **Notes layer** (evaluated as paradigm replacement — rejected;
    adopted as attachment layer): per-node markdown at `data/notes/<id>.md`
    for narrative documentation (what it does, gotchas, history) — prose
    that doesn't fit metadata fields. Wikilinks `[[node]]` parsed by a tiny
    collector into edges with `source: note` provenance (human-authored
    relationship claims). Rendered in the selection panel; git-native;
    curate surface can prompt for missing notes. Optional later tier:
    generator emits notes+frontmatter → **Quartz** static site (MIT) as a
    published prose-first knowledge base on GHE Pages, with backlinks/search
    — a doc-browsing skin over the same data, never the instrument.
    Obsidian/Quartz AS the graph: rejected — untyped links, no signals/
    lenses/trust grammar/ops/queries; the nightly fact layer would still
    need the whole collector pipeline, now emitting markdown.

---

## Consolidation protocol (for the session that cuts spec 1.6)

1. Do this at a phase gate or a dedicated docs session — never mid-task.
2. Read SPEC.md, FUTURE.md, PROGRESS.md, PLAYBOOKS.md, CLAUDE.md fully first.
3. Apply items by tag: [SPEC-*] into SPEC.md (rules into §1–§4, task text into
   the phase lists, §10 rewrite as noted) with a 1.6 changelog entry
   summarizing by group; [FUTURE±] edits to FUTURE.md; [PB]/[CLAUDE] edits to
   those files. [ASK-DAN]/[DAN] items: record in PROGRESS.md as open
   questions/human tasks — do not resolve them yourself.
4. Where an amendment contradicts existing text, the amendment wins; note the
   supersession in the changelog line.
5. Land as one commit series: `spec: 1.6 — <group>` per doc touched;
   PROGRESS.md updated in the same series.
6. Finish by DELETING this file in the final commit (`docs: consume
   AMENDMENTS.md into 1.6`). An empty staging file is the done-check; if any
   item can't be consolidated, it moves to PROGRESS.md as an explicit open
   question rather than remaining here.
