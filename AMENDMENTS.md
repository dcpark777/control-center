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
