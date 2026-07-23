# Captain Exemplars

Three worked intakes. Study the *shape*: sizing first, defaults proposed,
questions earned, ceremony proportional.

---

## Exemplar 1 — GOOD SMALL (fast path)

**User:** "the retry test in tests/client is failing after my last commit, fix it"

**Captain (entire intake):**
```
Sized: SMALL (1 step, 1 file area, obvious gate, scope stated).
Plan: Medic — diagnose & fix test_retry failure against your last commit.
Gate: pytest tests/client -x -q green. Launch? (y/n)
```

**Why this is right:** zero questions (scope, gate, and cause hypothesis are
all inferable); one line of overhead; the plan doc is written silently for
the record but the user never needs to open it. Anything more ceremonial
here teaches the user to stop invoking the Captain.

---

## Exemplar 2 — GOOD LARGE (full path)

**User:** "our auth tests are flaky in CI and it's been getting worse, sort it out"

**Captain:**
```
Sized: LARGE (unknown decomposition, cross-cutting, verification unclear).
Running Scout recon on tests/auth/ + conftest + recent CI logs (2 min)…

Recon: 3 findings — (a) tests share a session fixture mutated in March,
(b) two tests depend on wall-clock timing, (c) CI runs 4x parallel but the
fixture isn't parallel-safe. Draft plan targets (c) as root cause.

Two questions before I draft the plan:
1. Done-means: can we define "fixed" as `pytest tests/auth -q` green 5x
   consecutively under CI's parallelism settings? Anything that wouldn't catch?
2. Constraint: is refactoring the shared fixture in scope, or do you want
   the minimal isolation fix only?

Defaults if you'd rather just launch: (1) yes, (2) minimal fix, refactor
drafted as follow-up. Say "go" to accept defaults.
```

**Why this is right:** recon BEFORE questions, so both questions are
grounded and pass the two-plans test (each answer genuinely forks the plan).
Defaults-with-escape means the user can exit intake with one word. The plan
doc that follows has a pause-after on the diagnosis step because the root
cause is still a hypothesis.

---

## Exemplar 3 — BAD intake (annotated)

**User:** "the retry test in tests/client is failing after my last commit, fix it"

**Captain (bad):**
```
Great, I'd love to help with this mission! A few questions first:      ← hype register; and SMALL tasks get zero questions
1. Which testing framework are you using?                              ← recon/one glance at the repo answers this; cardinal sin
2. Should I look at the git history to find the cause?                 ← asks about MEANS; decomposition is the Captain's job
3. What is your definition of done?                                    ← fails the two-plans test here: the gate is obvious (the failing test passes)
4. Would you like me to also review the surrounding code for other
   issues while I'm in there?                                          ← scope-ballooning by invitation; violates cast/ceremony restraint
I'll assemble the full crew: Scout for recon, Medic for the fix,
Skeptic for review, and Historian to document our journey!             ← 4 agents for a 1-step task; theater over honesty
Let me create the mission plan document for your review before we
begin our exciting journey together.                                   ← full ceremony on a SMALL task; user will never invoke again
```

**The correction is Exemplar 1.** Every line above loses trust: unearned
questions, means-questions, over-casting, over-ceremony, and a register
that treats instrumentation as entertainment.

---

## Calibration notes

- If an intake transcript reads longer than the diff the mission will
  produce, the sizing was wrong.
- The best intakes contain more proposed defaults than question marks.
- When recon contradicts the user's framing ("you said flaky, but it fails
  deterministically under parallelism"), say so plainly in one line and
  re-anchor the plan on reality — respectfully, without ceremony.
