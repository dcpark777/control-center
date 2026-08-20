# cosmo — AMENDMENTS-2.md (staging, round 2) — RECONCILED against SPEC-ADDENDUM-A v2.1

Same rules as the consumed AMENDMENTS.md: staging, context-never-scope,
consolidate at a gate/docs session (apply by tag, amendment wins with
changelog note, [DAN] items recorded not resolved, delete when consumed).

STATUS SUMMARY (post-Addendum-A): #1 OPEN (independent track). #2 MOOT
(SPA frozen). #3 ABSORBED+REVISED by Addendum A. #4 SUPERSEDED (zero-plugin
decision). #5 RESOLVED (personal-first). #6 ABSORBED (V1.1/V2 control notes).
Only #1 remains actionable from this file.

1. [SPEC — scanner architecture] **The extraction flip.** OPEN — independent
   track, explicitly deferred by Addendum A header; not needed for V1.
   Deterministic-FIRST extraction is brittle for this fleet's idioms.
   Invert the tiers for IO edges:
   - Tier A (extraction): `claude -p` per repo, retrieval-scoped, reads the
     code and emits STRUCTURED findings — edges with {file, line, table,
     kind, confidence} — plus a prose "IO profile" note per repo (what it
     reads/writes and how, human-readable) written into the notes layer.
   - Tier B (verification, deterministic): cheap checkers validate each
     claimed edge — does file:line contain the sink/table string; does the
     table appear in the catalog whitelist (when present); does the dep
     exist in the manifest. Verified → confidence 0.95+; unverifiable →
     curate queue, never silently accepted.
   - Determinism moves from extraction to VERIFICATION + the commit gate.
   - Per-SHA caching unchanged; rescan = re-extract + re-verify, so
     campaignable=scannable still holds. Evidence pointers unchanged.
   - depscan stays deterministic-first; the flip applies to codescan/IO-edge
     extraction. Sunk parsers (sink visitor, sqlglot) become Tier B verifiers.
   - Addendum-A tie-in: emitted IO-profile prose lands in node pages'
     `## IO Profile` section (template slot already exists).

2. [SPEC — render] Graph view narrows to topology. MOOT — SPA frozen at
   1.25 by Addendum A decision record; reanimation trigger = demo date or
   Pages /map route.

3. [SPEC — export] `cosmo emit vault`. ABSORBED into Addendum A V1.1 with
   revisions that supersede this item's details: standard markdown links
   (NOT wikilinks) in the generated tier; zero plugins load-bearing (no
   cssclasses/Juggl styling, no Tasks-format dependency); full schemas per
   Addendum A §2. This item is historical context only.

4. [NOTE — vault plugin roster] SUPERSEDED by the zero-plugin decision
   (Addendum A non-goals). Dataview remains the sole pre-approved
   relief valve (DQL-only) if ad-hoc query friction proves weekly.
   Roster retained for reference: templater/tasks/advanced-tables/
   style-settings/kanban/quickadd/homepage were judged in-boundary;
   Shell Commands and MCP-transport plugins remain excluded regardless.

5. [GATE — P3 decision] Personal-first vs instrument-first. RESOLVED:
   personal-first, by Addendum A (vault = primary personal surface; SPA
   frozen; engineer tier = GitHub render; leadership = emitted brief +
   frozen demo; Quartz gated on hosting + named consumer).

6. [SPEC — control notes] Fleet Lens.md / Query Board.md with fenced
   blocks rewritten by `cosmo lens|query --emit-note`. ABSORBED into
   Addendum A (skeletons seeded at V1.1 --init; live rewrite in V2.5).
