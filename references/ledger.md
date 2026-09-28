# Ledger format

Path: `<scratch>/review-fix-loop/ledger.md`. Created before round 1; re-read at the start of every
round and after any context compaction; updated after reconcile and after each fix.

```markdown
# review-fix-loop ledger
target: PR #123 | A..B | branch feat/x vs main
repo: /abs/path   branch: feat/x   base: <sha>
spec: PR description (specAvailable: true)
options: reviewers=2 (opus-latest, gpt-latest) effort=high maxRounds=4 fixMinors=false
isolation: full | weakened (<reason>)
current_round: 2   state: reviewing | reconciling | fixing | done | escalated

## Rounds
| round | frozen head | worktree | reviewers | verified Major | fixer commits | tests |
|---|---|---|---|---|---|---|
| 1 | abc1234 | .../r1 | 1 | 2 | def5678 | pass |

## Findings
| id | sev | conf | axis | title | path:line | verdict | status | source | fix commit | attempts |
|---|---|---|---|---|---|---|---|---|---|---|
| R1-1 | Major | 0.9 | standards | nil deref on empty list | a.go:42 | confirmed | fixed | r1 | def5678 | 1 |
| R1-2 | Minor | 0.8 | standards | Unverified (single-reviewer...) | b.go:10 | demoted | open | r1 | - | 0 |

## Notes / escalations
- ...
```

Rules: `verdict` ∈ confirmed | demoted | rejected | unverified; `status` ∈ open | fixed | rejected |
accepted. Increment `attempts` each time a fixer is given the item. Never delete rows; flip-flops are
detected by a fixed/rejected row reappearing.
