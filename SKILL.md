---
name: review-fix-loop
description: Run a review→fix loop on a code change (PR number/URL, commit range, or current branch vs its base) until no verified Major finding remains. Fresh-context read-only reviewer subagent(s) review the whole frozen range each round, the main agent verifies and reconciles, and a separate fixer subagent fixes only verified Majors and commits. Host-agnostic (Claude Code, Codex, or sequential fallback). Use when asked to "review and fix until clean", "循环 review fix", "reviewer and fixer loop", "iterate review until no major issues", "keep reviewing and fixing", or "review-fix loop".
---

# Review → fix loop

Repeat **freeze → review → reconcile → fix** on a change until no verified **Major** remains.
Reviewer and fixer are different fresh-context agents. You, the main agent, are the
**orchestrator/reconciler**: you freeze, dispatch, verify, decide and report. You never review from
scratch and never edit code.

## Inputs (resolve before round 1)

- **Target**: PR number/URL (`gh pr view <n> --json baseRefOid,headRefOid,body`), a commit range
  `A..B`, or the current branch vs its base (`git merge-base <default-branch> HEAD`). Fail early if
  the range is empty or a ref does not resolve.
- **Spec**: PR description, linked issue, or the user's request. If none exists, `specAvailable: false`.
- **Options** (defaults): `reviewers=2` (one latest-Opus + one latest-GPT, independent, parallel),
  `effort=high`, `maxRounds=4`, `fixMinors=false` (only true if the user asked). The user may override
  any of these (e.g. single reviewer, specific model, lower effort).

## Host mapping

| Action | Claude Code | Codex | Neither available |
|---|---|---|---|
| Spawn fresh read-only reviewer / fresh fixer | `Agent` tool, `general-purpose`; several calls in one message = parallel | `spawn_agent` with no fork (`fork_context: false`, or `fork_turns: "none"` on the newer collaboration tools) | Play the role yourself, sequentially, clearing mental state; say in the final report that isolation was weakened |
| Wait for all | Results return in the same turn | `wait_agent` passing only `targets` | n/a |

Subagents cannot spawn subagents (Claude Code), so **all orchestration stays in the main agent**.

### Default models and effort

Unless the user specifies otherwise: **latest Opus** and **latest GPT** model, **effort high**, for
every role. Resolve "latest" at run time (aliases / host default), never pin a version string here.

- Reviewers (2, parallel): one on latest Opus, one on latest GPT. The same-engine one is a native
  subagent; the other engine runs read-only via shell:
  - Claude host: `Agent` with `model: "opus"` for reviewer A; reviewer B =
    `codex exec --sandbox read-only -m <latest GPT> -c model_reasoning_effort=high "<brief>"`.
  - Codex host: `spawn_agent` on the latest GPT with high reasoning effort for reviewer A; reviewer B =
    `claude -p --model opus --effort high --permission-mode plan "<brief>"` (read-only).
  If the latest-GPT id is unknown, check `codex` config/help or the host's model list; if either
  engine's CLI is missing or unauthenticated, fall back to two same-host subagents and note it in the
  final report.
- Fixer: one fresh subagent on the host's latest model (Opus on Claude, GPT on Codex), effort high.
- If the host's subagent tool cannot set effort, set it where possible (CLI flags) and note it.

Two engines doubles review tokens per round; that is the accepted default. With reviewers from both
engines, external-behavior claims can gate when both raise them (see `references/reconcile.md`).

## Ledger (compaction-proof state)

Keep `<scratch>/review-fix-loop/ledger.md` (scratch = the session scratchpad, else `$TMPDIR`).
**Re-read it at the start of every round.** Format and rules: `references/ledger.md`. It holds the
target, base, spec source, options, per round the frozen head, and every finding with
id, severity, confidence, verdict, status (open/fixed/rejected), fixing commit, fix attempts.

## One round

1. **Re-read the ledger.** Stop if `round > maxRounds` (escalate).
2. **Freeze.** `H=$(git rev-parse <head>)`; create a detached worktree
   `git worktree add --detach <scratch>/review-fix-loop/r<N> $H`. Reviewers read only there, so a
   moving branch cannot leak in. Record `H` in the ledger.
3. **Review.** Fill `references/reviewer-brief.md` (checkout path, exact range `BASE..H`, spec
   snapshot, prior findings + resolutions, full text of `references/review-policy.md` inline — the
   subagent sees nothing else). Spawn `reviewers` fresh read-only subagents in parallel; wait for all.
   Each reviews the **whole frozen range**, not just the last patch.
4. **Reconcile** per `references/reconcile.md`: verify each Major yourself in the round worktree,
   gate external-behavior claims on two-reviewer corroboration, merge duplicates, assign ids,
   update the ledger. No discovery sweep, no edits.
5. **Decide.**
   - No verified Major → **stop** (success). Remove round worktrees.
   - Anti-loop trigger hit (below) → **stop and escalate** to the user.
   - Else → step 6.
6. **Fix.** Fill `references/fixer-brief.md` with ONLY the verified finding list (plus verified
   Minors if `fixMinors`), not reviewer reasoning. Spawn one fresh fixer subagent (host's latest model, effort high) in the **real**
   working checkout (the branch, not the frozen worktree). It fixes only those items, runs the repo's
   own relevant test/build/lint commands, commits, and reports changes + test output. It never
   pushes, merges or force-pushes unless the user explicitly asked.
7. **Record** fixing commits and test results in the ledger; new head = branch tip; `round += 1`;
   go to 1.

## Stop and anti-loop rules

- Success: a round ends with zero verified Major.
- Escalate to the user (do not continue) when:
  - `maxRounds` (default 4) is reached with a Major still open;
  - the **same Major survives two fix attempts** (ledger `attempts >= 2` and re-verified open);
  - findings **flip-flop**: a Major marked fixed/rejected reappears, or a fix for A reintroduces an
    earlier B;
  - the fixer's **tests fail twice** (two rounds, or two consecutive attempts in one fixer report).
- Minor/Nit never block. They are listed in the final report.

## Final report (to the user)

1. Target, base, final head, rounds run, reviewers per round with model + effort used, and any fallback from the defaults.
2. Per round: frozen head, findings raised (id, severity, title), verdict after verification
   (confirmed/demoted/rejected), resolution (fixing commit or rejected reason). The **delta between
   rounds** is the key signal: what was fixed, what regressed, what was newly found.
3. Final verdict: `CLEAN` (no verified Major) or `ESCALATED` with the reason and the open items.
4. Remaining Minor/Nit (verified), with paths.
5. Tests/builds the fixer ran and their final status.
6. Isolation: `full`, or `weakened` (roles played sequentially in one context) and why.
7. Nothing was pushed or merged unless the user asked; list local commits created.

## References

- `references/review-policy.md` — the reviewer policy (axes, evidence, rating, output JSON).
- `references/reconcile.md` — orchestrator verification and merge rules.
- `references/reviewer-brief.md` — reviewer prompt template.
- `references/fixer-brief.md` — fixer prompt template.
- `references/ledger.md` — ledger format.
