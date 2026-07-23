# CREW — Mission Control for Claude Code

**Working name:** `crew` (alternatives to explore: Callsheet, Captain, Ensemble)
**Status:** v0 draft spec
**One-liner:** Summon a crew of named agents onto any task — plan it together in a document, watch it execute on a live board, steer it at any moment.

---

## 1. Vision

Claude Code is powerful but opaque: work happens in a scrolling terminal, plans live in the model's head, and multi-step tasks give the user no map, no progress, and no steering wheel short of interrupting. CREW supplies the missing layer for a *unit of work* (minutes to hours — "fix the flaky auth tests," "debug this Jenkins failure," "migrate this module"):

1. **A cast, not a tool.** Named, single-purpose agents (Scout, Skeptic, Medic, Historian, Janitor, Cartographer…) that get *staffed onto your problem* by a coordinating meta-agent, the **Captain**.
2. **A plan you negotiate.** The Captain interviews you, runs recon, and drafts a mission plan as a markdown document you edit and comment inline. Approval is a checkbox.
3. **Execution you can see and steer.** A live mission board (terminal blocks, statusline, TUI, HTML) rendered from one canonical state file. Pause, redo with critique, edit files yourself mid-flight, renegotiate scope.
4. **Just skills.** Distribution is a skill pack + agent files. No daemon, no server, no account, no hooks. `uvx`-trialable. Works in any editor or none.

Success is felt in one sentence: *"I always know what my agents are doing, why, and what they need from me."*

## 2. Positioning

**vs. BMAD (and methodology frameworks).** Same genus — named personas, plan-as-document, human gates — different species. BMAD is a *methodology you adopt*: an agile SDLC (analyst → PRD → architecture → stories → dev → QA) suited to greenfield builds over days or weeks, with org-chart-shaped agents (PM, Architect, Scrum Master) and documents as its main output. CREW is a *crew you summon*: invoked per-task on whatever you were already doing, capability-shaped agents (recon, review, diagnosis), and its differentiators live **after "approve"** — live observability, a six-state vocabulary, an append-only event log, checkpoint/redo with critique injection, anti-thrash limits, and inline plan negotiation. BMAD plans well; CREW shows you the work happening and hands you the wheel. (This paragraph, condensed, belongs on the README's first screen — the "isn't this BMAD?" question will be constant.)

**vs. Cascade-style session UIs.** Those streamline the minutes you spend *talking* to Claude in one workspace. CREW structures the *work itself*: plan, cast, gates, progress, redo. Complementary, not competing; CREW has no chat UI of its own — Claude Code's own session *is* the chat.

**vs. raw Claude Code.** CREW must never lose to raw CC at the small end (see Fast Path, §7.2). Its value begins the moment a task has more than one step or more than one thing worth verifying.

## 3. Design principles (the laws)

1. **Deterministic core, AI at the edges.** All state transitions, parsing, casting resolution, and file writes are deterministic code. AI plans, reasons, and produces work — it never *is* the ledger.
2. **One truth, many views.** `mission.json` + `events.log` are the sole runtime truth. Every surface (transcript blocks, statusline, TUI, HTML board, the plan doc's status mirrors) is a disposable projection of them. No two surfaces may ever disagree.
3. **The plan is the program.** Execution follows the compiled plan and nothing else. Deviations require a visible plan amendment. Agents are the runtime, not the author, of scope.
4. **Markdown for negotiation, JSON for execution, git for history.** Humans read/write markdown; machines execute schema-validated JSON; every revision of anything is a commit.
5. **Edits are commands, comments are conversation.** Direct changes to the plan are authoritative and obeyed; marked comment lines get answered inline. The Captain never guesses — ambiguity produces a question, not an assumption.
6. **The framework owns the protocol.** Crew members contribute personality and expertise only; status updates, checkpointing, interrupts, redo behavior, and artifact placement are injected by the Captain's harness at dispatch. A member *cannot* be written wrong.
7. **No required surface.** Every interaction must be possible with a terminal alone. Editor, browser, and TUI are enhancements. No specific editor may ever be required.
8. **Ceremony proportional to scope.** A ten-minute task gets a one-line plan and a "y". Full ceremony (doc, board, checkpoints) is earned by task size, never imposed.
9. **Honest theater.** The board shows real state including failure and pause — never animation for its own sake. Cast small: one agent for small jobs; the full ensemble must be warranted.
10. **Calm by default.** Interrupts only when the mission is blocked on the human. Everything else waits to be pulled.
11. **Doubt must be cheap.** Every gesture that expresses doubt — status, why, redo, revert — must cost less than the doubt itself. Friction on rejection quietly becomes pressure to accept; friction on questions quietly becomes distrust. This law outranks aesthetics.

## 4. Concepts & vocabulary

| Term | Meaning |
|---|---|
| **Mission** | One invocation of CREW on one goal, with its own directory, plan, state, log, and checkpoints. |
| **Captain** | The coordinating agent (a skill running in the user's interactive CC session). Interviews, casts, compiles, dispatches, reports, renegotiates. The steering wheel. |
| **Crew member** | A named single-purpose agent defined in one markdown file; runs as a CC subagent under the Captain's harness. |
| **Plan** | `mission-plan.md` (human source) compiled to `plan.json` (executable truth). |
| **Step** | One crew assignment with scope, acceptance criterion, and gates. The unit of checkpointing and redo. |
| **Gate** | A machine-checkable command whose exit status verifies a step (e.g. `pytest tests/auth -x`). |
| **Definition of done (DoD)** | Mission-level acceptance criteria, machine-checkable wherever possible; verified in the wrap. |
| **Checkpoint** | A git commit/snapshot after each step (and at declared review pauses). The unit of rewind. |
| **Mission states (exactly six)** | `queued ○` · `active ▶` · `done ✓` · `paused-for-you ⏸` · `failed ✗` · `redoing ↻` — same glyphs, colors, and names on every surface. |

## 5. Architecture

### 5.1 Runtime shape

- The **Captain is a Claude Code skill** invoked in the user's interactive session (`/mission <description>` or natural language trigger). It remains the user's live conversation for the whole mission.
- **Crew members are Claude Code subagents**: markdown files with YAML frontmatter (`name`, `description`, `tools`), auto-discovered from `.claude/agents/` (project) and `~/.claude/agents/` (user); project definitions take precedence on name conflict. CREW layers its own richer manifest on top (§10) and generates/validates the CC-compatible files.
- **A small deterministic helper CLI** (`crew`, stdlib-leaning Python) performs everything that must not be left to a model: plan compilation & validation, `mission.json` mutations, event appends, checkpoint commits, board rendering, `mission watch` TUI, lint, cleanup. The Captain calls it as a tool; the human can call it directly.
- **No hooks, no daemon, no server (v1).** Progress surfaces read files. The optional statusline integration uses CC's supported `statusLine` settings command (a script reading `mission.json`) — to be verified as permitted per environment; all other surfaces work without it.

### 5.2 Mission directory layout

```
.crew/missions/<mission-id>/
  mission-plan.md      # human negotiation surface (source)
  plan.json            # compiled executable plan (versioned)
  mission.json         # live state: steps, statuses, costs, active agent
  events.log           # append-only JSONL event log (the "why" record)
  artifacts/           # crew outputs: maps, reports, heatmaps, story
  board.html           # static rendered board (regenerated on change)
  worktree -> ...      # git worktree for code-touching missions
```

`.crew/` is gitignored by default except plans/logs if the team opts in. Every plan revision and checkpoint is a commit in the worktree/repo, tagged `crew/<mission-id>/…`.

### 5.3 Data flow

```
intake chat ──▶ mission-plan.md ──compile──▶ plan.json ──▶ Captain dispatch loop
   ▲                 ▲   │                                     │
   │            user edits/comments                    subagent (harnessed)
   │                 │   ▼                                     │
   └── clarifying blocks written back            status → mission CLI → mission.json + events.log
                                                              │
                                    transcript blocks · statusline · mission watch · board.html
```

---

## 6. File formats

### 6.1 The plan dialect (`mission-plan.md`)

A deliberately small markdown dialect. Only these constructs are meaningful to the compiler; everything else is prose and preserved untouched.

```markdown
# Mission: fix flaky auth tests            ^mission
- goal: auth test suite passes reliably in CI
- done means:
  - [gate] `pytest tests/auth -x -q` green 5x consecutively
  - [gate] no new dependencies (`git diff --stat requirements*`)
  - [human] Dan agrees flake root cause is explained in the wrap

## Step 1 — Scout: recon                   ^s1
- scope: tests/auth/, conftest.py, recent CI failures in artifacts/ci-logs/
- done means: written brief of suspected causes, ranked
- artifact: recon-brief.md

## Step 2 — Medic: fix                     ^s2
- needs: s1
- scope: minimal change to eliminate top-ranked cause
- done means: `pytest tests/auth -x -q` green
- gate: `pytest tests/auth -x -q`

## Step 3 — Skeptic: adversarial review    ^s3
- needs: s2
- scope: the step-2 diff AND retry logic in client.py
- done means: no critical findings, or findings triaged by Dan
- pause-after: yes            # declared human checkpoint

---
- [ ] **Approve plan and launch**          ^approve
```

Rules:
- **Anchors** (`^id`) are stable identities; reordering text never breaks references.
- **Step fields** are `key: value` lines from a fixed vocabulary (`scope`, `needs`, `done means`, `gate`, `artifact`, `pause-after`, `cast`). Unknown keys are a compile warning, not a guess.
- **Comments**: any line beginning `?>` is addressed to the Captain. It replies *indented beneath*, inside a `> [captain]` block, optionally offering checkbox options with their own anchors. Captain-authored text lives ONLY in `> [captain]` blocks and status-mirror sections — it never edits human prose.
- **Checkboxes** are the only state a human writes: launch, per-checkpoint resume, option selection, redo confirmation.
- **Status mirrors** (per-step one-liners the Captain renders back after transitions) are clearly marked read-only regions; editing them has no effect and triggers a gentle note.

### 6.2 Compilation

`mission compile` (deterministic) parses the dialect → validates against the plan schema → on success, writes `plan.json` and commits `plan vN`; on ambiguity/failure, the *Captain* posts a clarifying `> [captain]` block at the offending line and no new plan version exists. The compiler never repairs silently. The plan **freezes at launch**; subsequent edits compile into *amendment* proposals that take effect at dispatch boundaries (queued steps) or raise questions (active/done steps → "redo under new criteria? [ ]").

### 6.3 `plan.json` (excerpt)

```json
{
  "mission_id": "2026-07-23-auth-flake",
  "version": 3,
  "goal": "auth test suite passes reliably in CI",
  "dod": [
    {"kind": "gate", "cmd": "pytest tests/auth -x -q", "repeat": 5},
    {"kind": "gate", "cmd": "git diff --stat requirements*", "expect": "empty"},
    {"kind": "human", "text": "flake root cause explained in wrap"}
  ],
  "steps": [
    {"id": "s2", "member": "medic", "needs": ["s1"],
     "scope": "minimal change to eliminate top-ranked cause",
     "done_means": "pytest tests/auth -x -q green",
     "gates": [{"cmd": "pytest tests/auth -x -q"}],
     "pause_after": false, "max_attempts": 3}
  ]
}
```

### 6.4 `mission.json` (live state, excerpt)

```json
{
  "mission_id": "2026-07-23-auth-flake",
  "plan_version": 3,
  "state": "active",
  "active_step": "s2",
  "steps": {
    "s1": {"state": "done", "attempts": 1, "artifact": "artifacts/recon-brief.md",
            "started": "…", "ended": "…", "cost_usd": 0.14},
    "s2": {"state": "active", "attempts": 1, "started": "…", "cost_usd": 0.31}
  },
  "totals": {"elapsed_s": 412, "cost_usd": 0.45, "tokens": 88214},
  "awaiting_human": null
}
```

Cost, elapsed time, and a rough remaining estimate are first-class and shown on every surface.

### 6.5 `events.log` (append-only JSONL)

Every state transition is an event: `{ts, actor, step, from, to, reason, evidence?}`. `failed` events MUST carry an evidence excerpt (failing assertion, log lines) so "why" is attached, never hunted. The board, wrap story, and all "why is this X?" answers are projections of this log.

---

## 7. The Captain

### 7.1 Lifecycle: Brief → Recon → Plan → Approve → Execute → Wrap

### 7.2 Sizing & the Fast Path (Law 8)

On invocation the Captain sizes the task. **Small** (single obvious step, low blast radius): it proposes a one-step plan *inline in chat* — "Plan: Medic diagnoses & fixes; gate: `pytest -x` green. Launch? (y/n)" — a "y" launches; the plan doc is still written (for the record) but never requires opening. **Medium/large**: short interview (scope, constraints, DoD), then recon, then full plan doc. The interview is capped (≤4 questions) and every question must change the plan; anything else is skipped.

### 7.3 Recon before plan

For non-trivial tasks, Scout runs a bounded recon DURING intake (read-only, time-capped) so the draft plan is grounded in the actual code — "your auth tests import a fixture that moved; that's likely the real problem" — rather than confident fiction. Plan-after-recon is the default; the user can skip it.

### 7.4 DoD elicitation

The highest-leverage moment. The Captain pushes acceptance criteria toward machine-checkable gates ("done means: it works" → negotiated into a command, or explicitly marked `[human]`). Vague DoD is a compile warning surfaced before launch.

### 7.5 Casting

Deterministic matcher over member manifests' `recruit_when` hints + Captain judgment, with **cast restraint** enforced: the plan must justify each member; default is the minimum viable cast. Repo-local members outrank user-level ones (mirroring CC subagent precedence).

### 7.6 Dispatch loop

For each ready step: (re)read plan.json & mission.json → detect human edits since last checkpoint (diff worktree) → commit any as `human edit`, announce, and include in the briefing → wrap the member's prompt in the **harness** (§10.3) → dispatch as subagent → on return, run gates via `mission` CLI → transition state, append events, checkpoint commit → render surfaces → proceed, pause (`pause-after` / `paused-for-you`), or renegotiate (§9.4).

**v1 execution is sequential** with clean seams. True parallel fan-out (multiple CC processes across worktrees) is explicitly v2 — the spec commits to not implying parallelism the runtime doesn't deliver.


---

## 8. Interaction model & surfaces

### 8.1 The four gestures

**Glance** (statusline / board / badge) · **Decide** (tick a checkbox, answer an option block, "y" in chat) · **Steer** (chat to the Captain: pause, redo with critique, reprioritize) · **Edit** (change the plan doc or the code directly — both are first-class events).

Per Law 7, chat alone can do all of it: the Captain accepts every plan operation conversationally ("swap steps 3 and 4", "add a gate to step 2") and both routes converge on the same compiled plan.

### 8.2 Surfaces (all projections of mission.json + events.log)

1. **Transcript blocks** — after every transition the Captain prints a compact unicode board:
   `✓ Scout recon → recon-brief.md · ▶ Medic fixing (2m, $0.31) · ○ Skeptic · next pause: after s3`
   Free, zero-config, works everywhere. This is the guaranteed baseline surface.
2. **Statusline** — optional script for CC's `statusLine` setting: `⛑ auth-flake ▶ Medic 2/3 ██▓░ $0.45`. Environment-permitting.
3. **`mission watch`** — Rich/Textual TUI in a split pane: full board, gate sparkline, tailing events.log. v2: interactive keys (r=redo, p=pause) writing control events — the no-server path to a clickable board.
4. **`board.html`** — static local page regenerated on change (auto-refresh meta): the visual mission board with cast avatars, DAG, artifact links, cost. Read-only in v1. The shareable/demo face.
5. **Notifications** — even in v1: terminal bell + (where available) an OS notification when the mission enters `paused-for-you` or `failed`. A blocked mission must not wait unnoticed (audit finding).

### 8.3 UX affordances (Law 11 made concrete)

**First contact.** `mission init`: scan the repo, introduce the day-one cast by name, run a free time-capped Scout recon so the first output is insight about the user's actual code, not configuration. Bare `mission` (no args) answers "where was I?" — every open mission, its state, what's waiting on the human: the resumption brief as the default gesture.

**Plan stage.** Per-step cost/time estimates rendered before launch ("~8 min, ~$0.60") — informed greenlights over surprised retrospectives. Plan revisions render as diffs (v2→v3 shows what changed; never force a full re-read). Verb presets — `mission fix|review|upgrade|investigate <desc>` — pre-shape sizing and casting for common task shapes, shaving intake.

**Execution.** `mission status` includes the current tool action, not just the step (agent opacity is measured in seconds of doubt). `mission why <step>` prints the reasons behind any state straight from events.log. Notifications carry the decision itself ("Skeptic found 2 criticals in auth diff — review?") so the glance is often the interaction.

**Review sweep.** Diff hunks sort by Skeptic's risk annotations — scariest first, respecting the attention budget. Rejection is two keystrokes: one-key redo with canned critiques (too broad / missed edge cases / wrong approach / custom).

**Trust texture.** Attempt transparency: "take 2 of 3" always visible on every surface. `mission revert <step>` is a permanent, visible undo path — people steer boldly when the exit is marked. A Captain preferences file (`captain-prefs.md`, plain markdown, tunable and layered like everything else) accumulates standing defaults — "always minimal diffs," "never cast Historian unasked" — read at every intake so the tool stops re-asking what it should know about its Command.

### 8.4 Legibility rules (binding)

Six states, fixed glyphs/colors, plain language everywhere ("Skeptic is reviewing the diff (2m)"), progressive disclosure (glance-line → paragraph → full artifact/transcript), and review screens lead with the delta since your last look.

---

## 9. Execution semantics

### 9.1 Predictability
The compiled plan is the program (Law 3). Members may only act within their step's scope; the harness states this and the Captain enforces it at review. Any needed deviation → plan amendment event, visible before or immediately at occurrence, never silent.

### 9.2 Checkpoint & redo
Checkpoint commit after every step. Reject a step (chat or checkbox) → revert to its checkpoint → re-dispatch with the critique injected ("previous attempt rejected because: …"). **Anti-thrash:** `max_attempts` per step (default 3, incl. redos); on exhaustion the Captain stops, presents the attempt history and options (rescope / human takes it / skip / abort) — it never flails.

### 9.3 Interrupts & human edits
`pause` takes effect at the next tool boundary: the active member checkpoints partial work and yields; nothing is lost. Human file edits are detected at every dispatch boundary, committed as `human edit`, announced, and briefed to subsequent members ("Dan hand-modified the retry logic — respect it").

### 9.4 Renegotiation (mid-flight discovery)
When a step reveals the task is materially bigger/different, the Captain pauses and presents the discovery with re-scoped options: **expand** (amended plan), **split** (finish reduced scope now; draft a follow-up mission), or **abort**. Missions that end honestly at "bigger than we thought — here's the map" are successes; ballooning silently is forbidden.

### 9.5 Mission acceptance (the wrap)
"All steps done" ≠ "goal achieved." The wrap re-runs every DoD gate against final state, diffs outcome vs. DoD, and reports honestly, including partial delivery ("delivered; 2 of 3 criteria met; the third needs X"). Then Historian writes the story: narrative wrap from events.log + artifacts, linked from the board.

### 9.6 Resume, abort, cleanup
The Captain's CC session WILL die (context limits, crashes, closed laptops). `mission resume <mission>` is a first-class flow: a fresh Captain session rebuilds full context from plan.json + mission.json + events.log + checkpoints and continues — resumability is a designed property, not luck. `mission abort` tears down the worktree, parks artifacts, closes the log with an `aborted` event; no orphaned state.

### 9.7 Learning loop (v1 stub)
Post-wrap, a retro entry (what was redone and why, gate first-pass rate, casting notes) is appended to `.crew/learnings.md`. Additionally, whenever casting finds no member whose `recruit_when` matches a needed capability (the Captain falls back to a generic implementer), a `cast_gap` event is logged with a one-line description of the missing specialty. Future: feeds casting and recon briefs. Keeps the compounding story alive without new machinery.

### 9.8 Campaigns (fleet changes)

A **campaign** applies one change across many repos via: **pilot → validate → fleet.**

1. **Pilot.** One full mission on a representative repo, normal cast and ceremony.
2. **Recipe.** The **Tactician** (cast member) takes the pilot's plan, final diff, AND event log (redos, renegotiations, first-try gate failures locate where fleet variance will bite) and writes a **recipe**: a plan template in the standard dialect with variables (`{{connector_version}}`, `{{config_path}}`) plus two blocks — **preflight** (deterministic applicability checks: greps, version pins, file presence; no AI needed) and **variance notes** ("repos pinning <2.x need the shim step"). Same compiler, same gates — a recipe is just a parameterized plan.
3. **Sort.** Preflight runs across the fleet and sorts repos into *standard* (recipe applies) vs *variant* (flagged for adapted missions with fuller cast). This converts "10 unknown repos" into "7 stamps and 3 real missions" before expensive work starts.
4. **Validate.** A recipe from n=1 is provisional — generalizing from one pilot is guessing. Repo #2 runs as a validation mission; the Tactician revises the recipe from the deltas. Only then does the fleet get stamped.
5. **Fleet.** Standard repos run as fast-path missions instantiated from the recipe (sequential in v1; parallel worktrees v2). External actions (PR creation, pushes) are per-action approvals batched into a single **review sweep** at the end — all diffs + gate results as cards, one decision each, never N interruptions mid-run.

Campaign state is a thin layer over missions: `campaign.json` tracks per-repo mission ids and preflight results; everything else reuses mission machinery.

---

## 10. Crew members & extensibility

### 10.1 Manifest (one file = one member)

```markdown
---
name: skeptic
glyph: "🔎"        # board identity
color: "#d97706"
purpose: adversarial review of diffs before humans see them
recruit_when: a change exists that hasn't been challenged; risk of shortcuts, missing tests, hidden coupling
tools: Read, Grep, Bash(pytest*)          # allowlist — enforced, a security boundary
gates_default: []
artifact: risk-review.md                   # declared output; board renders generically
---
You are the Skeptic. You assume every diff hides a shortcut…
(voice + expertise ONLY — no protocol instructions)
```

### 10.2 Discovery & layering
Presence is registration: `~/.crew/members/` (personal) + `./crew/` (repo-local, versioned with the code, takes precedence). `mission crew sync` generates the CC-compatible subagent files into the appropriate `agents/` directories.

### 10.3 The harness (Law 6)
At dispatch the Captain wraps every member prompt with the standard protocol: step scope & done-means, status reporting via the `mission` CLI, checkpoint discipline, interrupt behavior, redo-critique handling, artifact placement, and the scope fence. Authors write personality; the framework injects integration — a member cannot break the board.

### 10.4 Creation & safety
Three doors: copy-and-edit; `mission crew new` (validated scaffold); the **Workshop** member, which interviews you and writes the file. `mission crew lint` validates the manifest, runs a smoke mission, and performs a **safety pass**: community-shared members are a prompt-injection vector, so lint flags suspicious instructions and the harness's tool allowlist is enforced at runtime (a security feature, stated as such). Sharing = a file: gist, PR, later `mission crew install`.

### 10.5 Starter cast — an earned roster, not a shipped ensemble

**Day one (3):** Scout (recon/brief) · Medic (diagnose/fix from logs & failures) · Skeptic (adversarial review). These cover the daily 80%: investigate, fix, verify.
**On first campaign:** Tactician (recipe generalization, §9.8).
**On demand thereafter:** Historian (docs/ADR/story — when wraps start mattering) · Workshop (member creation — when making members becomes common) · Janitor · Cartographer/atlas · Story Mode (v2 crew passes).
Roster restraint mirrors cast restraint: every shipped member must answer a recurring real need, because unused members are noise in every casting decision.

**The roster grows itself (human-approved).** Gap detection is deterministic: `cast_gap` events (§9.7) plus retro patterns (the same ad-hoc instruction appearing across missions, recurring `[human]` criteria that a specialist gate could automate, high redo rates clustered in one category). `mission retro` aggregates these and proposes: "4 missions this month needed secret-handling review — draft a member for it?" On approval, the Workshop interviews, drafts the file, and `mission crew lint` smoke-tests it before it joins the roster. The inverse is also surfaced, never auto-acted: members uncast for N missions get flagged for retirement as a suggestion only. Usage → gap signal → drafted proposal → human decision: the crew is self-extending, but Command signs every hire.

---

## 11. Observability & metrics

Every surface answers: what's happening, why, what it has cost (elapsed, $, tokens), roughly how much remains, and what needs the human. Framework health metrics (measured from events.log, local-only): redo rate · gate first-pass rate · ceremony overhead vs. task size · resume success rate · notification regret (was this interrupt worth it). These are the claims "it helps you deliver" is tested against.

## 12. Security & environment constraints

- All AI operates on local data only; crew members get no network. Any external action (push, ticket comment, API call) surfaces as an explicit approve/deny to the human; the deterministic side executes approved actions.
- No Claude Code hooks; CC state is observed passively (files). Statusline is optional settings config, verified per environment.
- No telemetry, no accounts. Observes artifacts, not behavior.
- Tool allowlists in manifests are enforced boundaries (supply-chain defense for shared members).

## 13. MVP cutline

**v1 (the demo IS the onboarding):** Captain skill (fast path + full path, recon, DoD elicitation, captain-prefs) · plan dialect + compiler + amendments (plan-diff rendering) · sequential dispatch with harness · six states, checkpoints, redo-with-critique (canned critiques), anti-thrash · resume/abort/revert · transcript blocks + board.html + bell/notification · starter cast (Scout, Medic, Skeptic) · `mission` CLI (init, bare-`mission` where-was-I, compile, status incl. current action, why, render, crew new/lint, resume, abort, revert) · cost/elapsed estimates · learnings stub + cast_gap events.
**v2:** `mission watch` interactive TUI · true parallel fan-out across worktrees · Workshop + Janitor + Cartographer/atlas + story mode · statusline polish · `mission crew install` sharing · candidate-3 daemon integration (missions as loops in the working-memory ledger).

## 14. Open questions — status

Resolved since v0 draft:
1. **Name — RESOLVED.** Vocabulary: mission/crew; orchestrator is the **Captain**, the user is **Command**; entry point is the `/mission` skill; CLI binary is `mission` with crew management under it (`mission crew new|lint|sync`). CrewAI collision noted; acceptable for a personal project, revisit only if public.
4. **Approval authority — RESOLVED.** Last event wins, both logged; every launch/resume echoes one line of exactly what is being approved. Codified as the "Approval ambiguity" row of the Captain codex §11.
5. **Harness visibility — RESOLVED.** Fully auditable: `mission harness --show` prints the assembled wrapper verbatim (base + local layer + filled variables for a given step). Follows from the tunability rule — no hidden prompts anywhere.

Open, and answerable only empirically (dogfooding milestones, not decisions):
2. **Statusline permissibility** in restricted environments — test at install time per environment; design already treats it as optional garnish.
3. **Plan-doc watching without hooks** — v1 ships re-read-at-boundaries + explicit "recompile"; the first week of real use decides whether that ergonomic suffices or the v2 helper process gets pulled forward.
