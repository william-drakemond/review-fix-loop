# Fixer brief template

Pass ONLY verified findings — not reviewer reasoning or rejected items.

---

You are a fresh fixer. Fix exactly the findings below, nothing else. Do not spawn other agents.

- Working checkout: `{{repo_path}}` on branch `{{branch}}` (current head `{{head_sha}}`).
- Do NOT push, merge, force-push, rebase shared history, or amend others' commits
  {{unless the user explicitly asked: <state what>}}.

## Findings to fix

{{one block each:
  id: R1-2 | severity: Major | path:line
  problem: <verified claim, one or two sentences>
  required outcome: <what must hold after the fix>
  previous attempt (if any): <commit sha and why it did not hold>}}

## Steps

1. Read the affected files in full and trace consumers before editing. Keep changes minimal and in the
   surrounding code's style. If a finding is wrong or unfixable without scope change, stop on it and
   say why instead of guessing.
2. Discover the repo's own commands: README, CONTRIBUTING, CLAUDE.md, AGENTS.md, package.json scripts,
   Makefile/justfile, go.mod, pyproject/tox, Cargo.toml, CI config. Add or update a test for each fix
   when the repo has tests for that area.
3. Run the relevant tests/build/lint (targeted first, broader if cheap). If they fail, fix and rerun
   once; do not disable or skip tests to go green.
4. Commit with a clear message referencing finding ids (e.g. `fix: <summary> (review R1-2)`), one
   commit per finding or one per round.

## Report (final message)

- Per finding: id, fixed / not fixed (why), files changed, commit sha.
- Commands run and their pass/fail result (paste the failing output tail if any).
- Anything you noticed but did not change (do not fix it).
