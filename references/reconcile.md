# Reconciler rules (orchestrator)

You verify and merge reviewer reports. You do **not** run another review sweep, add findings no
reviewer reported, or edit code. Read only what is needed to confirm or refute claims, inside the
round's frozen worktree. Stay static: no installs, no test runs.

## 0. Check coverage first

Before using a report, compare its `evidenceChecklist.filesRead` with the frozen range's changed-file
list. A report that skipped changed files, or that says it only re-checked prior fixes, is
**non-compliant**: re-dispatch that reviewer slot once with a brief naming the missed files. Never
count a partial review as a clean vote.

## 1. Verify every Major

Classify its central claim:

- **Repo-verifiable** (about the diff, call sites, types, contracts — anything a read/grep in the
  checkout can settle): verify it yourself. A reviewer's confident prose is a claim, not proof.
  Confirmed → keep; disproved → `rejected` with reason; only a narrower issue holds → demote to Minor
  and state the narrower claim.
- **External-behavior** (third-party API, library runtime, platform or infrastructure semantics not
  checkable in the repo): gates only if **two independent reviewers** each list it in their own
  `findings[]`. Otherwise demote to Minor, keep it, and prefix the body with
  `Unverified (single-reviewer claim about external behavior): `.
  With a single reviewer (user override) such claims therefore never block; say so in the report so the user can decide.

Re-apply the four-part Major proof (obligation, reachable failure, evidence, blocking impact) to the
merged evidence. Severity comes from verified impact, not the highest reviewer rating; agreement and
confidence do not replace the proof.

## 2. Merge

- Union findings, dedupe **semantically** (same defect, two wordings = one). Same line ≠ same defect.
- Record `source`: which reviewers independently raised it (only from their `findings[]`).
- Keep the reviewer's `axis`. If `specAvailable` is false, drop all spec findings.
- Assign your own confidence after verification; drop anything < 0.7.
- Match against the ledger: a finding equal to a previously `fixed` or `rejected` item is a
  **regression/flip-flop** — flag it (anti-loop) rather than silently reopening; an equal `open`
  item increments nothing until the fixer attempts it again.
- Give new findings ids `R<round>-<n>`; keep existing ids for recurring items.

## 3. Decide

- Zero verified Major → round clean → stop.
- Any verified Major → fixer, unless an anti-loop rule in SKILL.md fires.
- A specific material failure you could not verify: record it as `unverified`, report it; do not let
  a generic hardening concern hold the loop open.

## 4. Update the ledger

Write each finding's severity, confidence, verdict (`confirmed|demoted|rejected|unverified`), status,
and source before dispatching the fixer.
