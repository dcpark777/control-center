# cosmo — instructions for Claude Code

cosmo is a platform fleet map: collectors → git-committed JSON snapshots → graph.json
→ rendered views + query layer. You are building it against a written spec.

## Every session, in this order
1. Read `SPEC.md` (§0 is the execution protocol — it governs, not this file).
2. Read `PROGRESS.md`. Work the first unchecked task in the current phase.
3. Update `PROGRESS.md` **in the same commit** as the work it describes.
4. Stop at `⛔ GATE` markers and wait for Dan. Ask `❓ASK-DAN` items; never guess them.

## Hard rules
- `FUTURE.md` is context, never scope. Do not build anything from it.
- `PLAYBOOKS.md` governs recurring maintenance: consult the matching playbook before
  adding/renaming schema keys or fields, adding detectors, collectors, or node types,
  or renaming node IDs.
- `docs/mockup.html` is a design artifact. Read it for look/feel; never edit it,
  never copy its code.
- **Data safety:** write only under `$COSMO_DATA_DIR` (default `./data`). Never modify
  a prior day's snapshot directory. Never rewrite git history. `annotations/` is
  human-authored and unregenerable — write-path changes require fixture tests first.
- **Binding schema rules (SPEC §4):** node IDs are permanent (renames go through
  aliases in `collectors/config.yaml`); signal/meta keys are deprecate-and-add only
  after the P1 gate.
- Python stdlib first. Any new dependency needs a written justification in
  PROGRESS.md before it's added.
- Tests never call `claude -p` or live APIs — fixtures and canned responses only.
- `make test` green before committing anything that touches `graph/` or `schema.py`.
- Per-repo special cases go in `collectors/config.yaml`, never hardcoded.

## Commits
Small, one concern each, imperative subject, e.g. `collectors: add mend severity
parse`, `progress: P1 collector contract done`.
