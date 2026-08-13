# cosmo — PLAYBOOKS.md

Recipes for recurring maintenance operations. Consult the matching playbook **before**
changing schema keys, metadata fields, detectors, collectors, or node types. These
encode the binding rules (SPEC §4) as procedures — follow them exactly; deviations get
flagged in PROGRESS.md, not improvised.

Format per playbook: when to use → steps → guardrails → done-check.

---

## PB-1 · Add a metadata field to a node type

*Use when:* a new per-node attribute is wanted (e.g. `business_component_id`).

1. **Name it once, carefully** — snake_case, full words. Renames later cost PB-2.
2. Add it to the node type's expected-meta list in `graph/schema.py` (this recruits
   completeness scoring, `missing · + add` panel rows, and the curation queue).
3. Choose the source:
   - **Manual:** a YAML map in `data/manual/<field>.yaml` (`repo:x: VALUE`) + a
     ~20-line collector emitting `source: manual`, `as_of` = file mtime.
   - **Derived:** extend the relevant existing collector.
4. Run `make all`; the field flows to panel, filters, export, chat, Pages with zero
   renderer work (unknown meta renders generically by design).

*Guardrails:* URLs go under `meta.links.*` (auto-buttons), never loose fields. Scalars
for facts; one object level for families; a value growing multiple attributes and its
own identity is a **node**, not nested meta (see PB-5).
*Done-check:* field appears in a repo panel; fleet completeness dropped; export shows
the column.

## PB-2 · Rename or migrate a key

*Use when:* a signal/meta key must change after the P1 gate (binding rule: keys are
deprecate-and-add; committed snapshot history is NEVER rewritten).

1. Add a dated entry to `KEY_ALIASES` in `graph/schema.py`:
   `{"old.key": ("new.key", "renamed YYYY-MM-DD")}`. Graph build and trend extraction
   normalize historical reads through this map. (First use builds the plumbing —
   [spec-1.5].)
2. New snapshots write only the new key. If external consumers can't flip at once,
   dual-write both keys for a stated deprecation window, then drop the old.
3. **Sweep checklist** — keys hide beyond snapshots:
   - expected-fields list in `schema.py`
   - saved views (serialized filter ops) — prefer teaching the filter evaluator the
     alias map over rewriting stored ops (also covers chat-authored filters)
   - export column configs
   - skill/prompt docs naming the key
4. Golden-file test spanning the boundary: one old + one new snapshot → one
   continuous trend line.
5. Spec changelog entry with rationale; land as a single commit.

*Guardrails:* meta keys are cheap (today's dir is mutable), signal keys drag trend
continuity, node IDs are PB-6 — a different procedure.
*Done-check:* trends continuous across the rename date; grep for the old key returns
only the alias entry and history.

## PB-3 · Add a codescan detector

*Use when:* a new data-IO pattern should become edges (Kafka topics, Delta paths,
REST calls) or a scan-on-demand rule proved out and gets promoted.

1. New detector module in the codescan registry: match rule (imports/AST/regex) →
   edges `{kind, source: codescan-ast, confidence: 1.0}`. Edge kinds are open strings
   — no schema or renderer change.
2. Fixtures first: at least one true-positive and one near-miss repo sample; detector
   ships with its golden tests.
3. Per-repo weirdness goes in `collectors/config.yaml` hints, never hardcoded.
4. If precision can't reach ~100% deterministically, don't ship it half-right — route
   ambiguous files to tier 2 (`claude -p`, `source: codescan-ai` + confidence) or
   leave it as a gated scan-on-demand rule until it earns promotion.
5. Bust the scan cache for affected repos (cache keys on commit SHA + detector-set
   version — bump the version).

*Guardrails:* trust over coverage — two detectors that are never wrong beat ten at
80%. Codescan is data-IO only; code structure belongs to the graphify adapter (P5).
*Done-check:* new edges carry correct provenance; legend/tooltip render them; golden
tests green; a known-negative repo gains no edges.

## PB-4 · Add a collector (new data source)

*Use when:* a new signal or metadata source (new API, manual file, CI system).

1. One module in `collectors/`, independently runnable, emitting JSON keyed by node
   ID into `snapshots/<today>/<name>.json`.
2. Honor the contract: nonzero exit on failure; `_status.json` (ok/failed/partial +
   error) beside output; `--only <node-ids>` / `--source` scoping; never blocks other
   collectors.
3. Every emitted signal carries `source` and `as_of`. What it can't determine is
   **absent or `unknown`** — never a stale value passed as current, never a default.
4. Golden-file tests from recorded (scrubbed) responses; tests never hit live APIs.
5. Register freshness display; a failing collector must surface as the degraded
   banner + unknown states, not silence (kill it mid-run once to verify).

*Guardrails:* secrets via env only, never echoed in status or logs.
*Done-check:* `make all` with the collector down still renders, visibly degraded.

## PB-5 · Add a node type

*Use when:* a new kind of entity earns identity (`ticket:`, `team:`, `component:`,
`job:`) — including promoting a meta field that grew multiple attributes.

1. Decide the **join key first** — how instances link to existing nodes (label
   convention, mapping file, derivable field). This is the real work; conventions are
   cheap now, backfills are not.
2. ID namespace: `type:identifier`, lowercase, permanent (PB-6 governs renames).
3. Expected-meta list for the type in `schema.py` (completeness works day one).
4. Collector per PB-4; edges use open `kind` strings.
5. Renderer: nothing required (generic styling); add a bespoke style only once the
   type is proven.
6. If promoting from meta: keep the scalar field through a deprecation window
   (PB-2), emit nodes + `belongs_to` edges alongside, then retire the scalar.

*Guardrails:* **teams, never individuals** — person nodes, especially with activity
data, violate the fleet-not-people rule (SPEC §1). Individuals are at most contact
info on a team node.
*Done-check:* new nodes render, link, filter, and export without renderer changes.

## PB-6 · Rename a node ID

*Use when:* a source renames (GitHub repo rename) or an ID was minted wrong.

1. Source renames: **the node keeps its ID** — add the new source name to the alias
   entry in `collectors/config.yaml`. Done; no migration.
2. True ID change (rare, deliberate): rename map applied atomically in ONE commit
   across everything holding IDs — `graph/layout.json`, `annotations/` (files and
   `target_edge` strings), the audit log (append a rename record; never rewrite),
   saved views, manual data files. Keep the old ID as an alias so stale references
   resolve.

*Guardrails:* increasingly expensive once other people's links/decks hold IDs —
prefer the alias, always.
*Done-check:* search finds the node under both names; annotations and layout survived;
no orphan edges.

---

**Maintenance of this file:** when a session performs one of these and hits a step
that's wrong or missing, fix the playbook in the same commit as the work — playbooks
that drift from practice are worse than none.
