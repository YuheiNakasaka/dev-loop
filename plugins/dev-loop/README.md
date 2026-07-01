# dev-loop (plugin)

A **plan + state + `/step`** loop-engineering harness for Claude Code. Two files are the source of truth instead of the chat history, so you can `/clear` between steps and resume:

- **`.claude/dev-plans/<date>-<slug>.md`** — the spec: an ordered list of steps (goal / success criteria / verification / dependencies). Written fresh each `/plan-init` run (never overwritten); the active plan is `dev-state.json`'s `planDoc`.
- **`.claude/dev-state.json`** — the ledger + config: per-step status, gate results, learnings, and all settings (branch, gate commands, project name). **Git-ignored / local (per-developer)** to avoid merge conflicts; progress is tracked via committed plan files + `Step N` commit history.

All configuration lives in `dev-state.json`, so the commands are completely project-agnostic.

## Commands

| Command | What it does |
|---|---|
| `/plan-init [goal · spec · review doc]` | Scaffolds a new `.claude/dev-plans/<date>-<slug>.md` (never overwritten) + a local `.claude/dev-state.json` (git-ignored), auto-detects quality gates, records a baseline, and sets up the review gate. |
| `/step [id]` | Runs one step: check deps → implement → run gates → **hierarchical review gate** → update state → one commit per step. No arg = next pending. |
| `/step-review [id]` | Read-only: runs only the review gate against the working tree and reports the verdict. No edits, no commit, no advance. |
| `/plan-status` | Read-only progress view (incl. review verdicts and blocked steps' human tasks). |

> A project-local command of the same name shadows the plugin's; invoke explicitly with `/dev-loop:step` if needed.

## Hierarchical review gate

Each `/step` ends with a **self-contained, hierarchical, iterative review** (no external plugin needed): perspective reviewers (`dev-reviewer`, breadth) in parallel → a meta-reviewer (`dev-review-meta`, *review of the review*) that adjudicates against the step's goal/scope → auto-fix & re-review for several rounds until it converges → the refined verdict drives the next action (commit & advance / leave **`blocked`** for human-only fixes / spin out-of-scope findings into new steps). Configured via `dev-state.json`'s `review` block; set `enabled: false` to keep the classic behavior.

## Reference

- Protocol, `dev-state.json` JSON schema, the *implement-vs-gate* framework, and the review-gate protocol: [`skills/dev-loop/SKILL.md`](./skills/dev-loop/SKILL.md)
- Built-in review engine: [`agents/dev-reviewer.md`](./agents/dev-reviewer.md), [`agents/dev-review-meta.md`](./agents/dev-review-meta.md), perspective catalog [`skills/dev-loop/assets/review-perspectives.md`](./skills/dev-loop/assets/review-perspectives.md)
- Templates: [`skills/dev-loop/assets/`](./skills/dev-loop/assets/)

For install instructions and the full write-up, see the [repository README](../../README.md).
