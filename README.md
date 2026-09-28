# review-fix-loop

An [Agent Skill](https://agentskills.io) that runs a **review → fix loop** on a code change until no verified **Major** finding remains. It works in Claude Code and in Codex.

- **Reviewers**: new read-only subagents with a fresh context every round. They review the whole diff against the base, not just the latest patch.
- **Orchestrator** (the main agent): verifies every Major against the code itself and merges duplicates. It never reviews from scratch and never edits code.
- **Fixer**: a separate subagent. It gets only the verified findings, runs the repo's own tests, and commits locally. It never pushes or merges.
- **Stop rule**: the loop stops when a round has no verified Major. It hands back to you after 4 rounds, when the same Major survives two fix attempts, when findings flip-flop between rounds, or when tests fail twice.
- **State**: kept in a ledger file, so the loop survives context compaction.

## Default models

Two reviewers run in parallel, one on the latest Opus and one on the latest GPT, both at high effort. The reviewer from the other engine is invoked through its CLI (`codex exec` or `claude -p`). If that CLI is missing, the skill falls back to two same-host subagents and says so in the final report. You can override the models, the effort, or the number of reviewers when you invoke it.

## Install

```bash
git clone https://github.com/william-drakemond/review-fix-loop ~/.agents/skills/review-fix-loop
# Claude Code also reads ~/.claude/skills:
ln -s ~/.agents/skills/review-fix-loop ~/.claude/skills/review-fix-loop
```

## Use

```
/review-fix-loop PR #123
/review-fix-loop main..feature-branch  reviewers=1 maxRounds=3
```

Or ask in plain words, e.g. "review and fix this PR until there are no major issues".

## Layout

| File | Purpose |
|---|---|
| `SKILL.md` | Loop procedure, host mapping, default models, stop rules, final report |
| `references/review-policy.md` | Reviewer policy: two review axes (built right / right thing), evidence rules, Major/Minor/Nit, JSON output |
| `references/reconcile.md` | How the orchestrator verifies and merges findings |
| `references/reviewer-brief.md`, `references/fixer-brief.md` | Prompt templates |
| `references/ledger.md` | Ledger format |

## License

MIT
