# PIVOT RUNBOOK — cosmo → command center (work machine)

## Files to transfer (this kit)
1. SPEC-ADDENDUM-A.md        (the v2.1 phased spec — save next to SPEC.md)
2. AMENDMENTS-2.md           (reconciled; skip if an identical/older copy already
                              exists repo-side — if one exists, REPLACE it with
                              this reconciled version so statuses match the addendum)
3. This runbook              (optional to transfer; it's for you, not Claude Code)

## Snippet — paste into SPEC.md (top, under the version line), commit with the addendum
```
## Decision record — 2026-08-2x (surface pivot)
Primary personal surface = emitted Obsidian vault (SPEC-ADDENDUM-A.md governs).
SPA frozen at 1.25 (tag: spa-1.25-frozen); reanimation trigger: a scheduled demo
or a Pages /map route. AMENDMENTS-2 #5 resolved personal-first; #2 moot; #3/#6
absorbed into Addendum A; #4 superseded (zero-plugin); #1 (extraction flip)
remains open as an independent track.
Repo hygiene standing rule: tag-then-prune — dormant code gets a tag if worth
naming and is deleted from HEAD regardless; HEAD = live system.
Vault = all human writes + all rendered output; cosmo repo = code + engine
config only.
```

## Order of operations

### Step 0 — Transfer + commit (5 min, no Claude Code needed)
- Copy the two files into the cosmo repo root (next to SPEC.md).
- Paste the decision-record snippet into SPEC.md.
- `git add -A && git commit -m "spec: addendum A (command center pivot) + reconciled AMENDMENTS-2 + decision record"`

### Step 1 — Session V0 (one evening; you drive, Claude Code assists)
Prompt:
```
Read SPEC-ADDENDUM-A.md phase V0 only. Execute it with me in the loop:
1) Run a full-history secrets scan (prefer detect-secrets or gitleaks from
   Artifactory; if unavailable, targeted `git log -p` greps for tokens,
   passwords, connection strings). Show me findings before any rewrite.
2) Report whether data outputs (graph.json, data/, scan artifacts,
   signals-history) are committed anywhere in history, and repo size
   offenders (venvs, large files). Propose a filter-repo plan if needed —
   do not execute a history rewrite without my explicit go.
3) Tag-then-prune: `git tag spa-1.25-frozen`, then remove the SPA directory
   from HEAD in one commit whose message points at the tag and the SPEC
   decision record. List any other dormant experiment dirs as candidates —
   I decide which get pruned.
Do not push anything; I will create the remote and push manually.
```
Then you: create the internal GitHub repo (most restrictive visibility
available — check the dropdown), add remote, push, approve.
Also this week, in parallel (Slack, not Claude): ask platform/appsec about
Mend programmatic access (export/API/token) for vuln counts.

### Step 2 — Session V1.0 (recon; read-only gate)
Fresh session. Prompt:
```
Read SPEC-ADDENDUM-A.md in full, then SPEC.md and AMENDMENTS-2.md.
Execute phase V1.0 ONLY — the reconciliation. Do not write any
implementation code and do not modify any existing files.
Produce V1.0-REPORT.md exactly per the spec: current spec version; built
overlaps (emit code, notes layer, `cosmo today`, Mermaid export, CLI/query
state, config.yaml contents that would migrate to vault meta/config/);
graph.json schema inventory vs Addendum A §2 with a per-gap estimate
(team, resolved-deps tier, Mend criticals, per-signal as_of, node ID
conformance); tool availability (jinja2, typer, jsonschema, packaging);
and any conflicts between the addendum and repo state with proposed
resolutions. Flag anything estimated over one session. Then stop.
```
Then you: read the report. Decide vault path/layout and the scope of V1.2
gap work. (Optionally bring the report back to chat for a second pass.)

### Step 3 — Sessions V1.1 (build; ~4-6 sessions)
One session per cluster, roughly in this order (respects dependencies):
  a) predicate engine + classifier pass + meta/config ingestion (+ tests, semver fixtures)
  b) minimal meta ingestion + merge precedence (F5) + provenance plumbing
  c) emit core: templates + determinism + ownership guard + atomicity + orphan deletion
  d) --init scaffolding + CLAUDE.md/USING/CONTRACT + dashboards (Home/README)
  e) --fast + CLI agent-contract pass + doctor + link-integrity report
Session-start prompt template:
```
Read SPEC-ADDENDUM-A.md §0.5 (foundations) and phase V1.1. Also read
V1.0-REPORT.md. Implement ONLY this cluster: <cluster from list>.
Use plan mode first for anything touching emit ownership (F3) or merge
precedence (F5) — show me the plan before writing code.
Commit per completed checkbox; check the box in SPEC-ADDENDUM-A.md in the
same commit. Run the relevant Gate-A fixtures before declaring a checkbox
done. Stop at the end of the cluster.
```

### Step 4 — Session V1.2 (data gaps per the report + your bootstrap hour)
- Claude Code: close collector gaps scoped in V1.0-REPORT (resolved-deps
  wiring; Mend collector if access landed, else interim curated field).
- You (with --fast open in a terminal and Obsidian open):
  1) Write meta/config/classifiers.md from your own heuristics (seed file
     has worked examples). Run --fast; review the vault git diff; use
     `cosmo classify --explain <repo>` on surprises.
  2) Work the Needs-curation queue on Home.md: team meta files for the
     fleet. --fast as you go.
  3) Set up bookmarks from the seeded views; adjust; save workspace layout.

### Step 5 — Gate A + verdict clock
- Claude Code runs the full fixture list (Gate A a-j); you spot-check the
  seven canonical queries by hand in Obsidian.
- Then STOP building. Live in it two working weeks. Keep the tally.
  Kill criteria are written in the spec — hold yourself to them either way.

## Deliberately NOT in this runbook
V2/V3/V4 prompts (written after the verdict, not before); Quartz; SPA
physics; anything Claudian-side beyond what --init seeds (Claudian setup
is independent and already done per your plugin decision).
