# dev-loop (plugin)

A **plan + state + `/step`** loop-engineering harness for Claude Code. Two files are the source of truth instead of the chat history, so you can `/clear` between steps and resume:

- **`.claude/dev-plan.md`** — the spec: an ordered list of steps (goal / success criteria / verification / dependencies).
- **`.claude/dev-state.json`** — the ledger + config: per-step status, gate results, learnings, and all settings (branch, gate commands, project name).

All configuration lives in `dev-state.json`, so the commands are completely project-agnostic.

## Commands

| Command | What it does |
|---|---|
| `/plan-init [goal · spec · review doc]` | Scaffolds `dev-plan.md` + `dev-state.json`, auto-detects quality gates, and records a baseline. |
| `/step [id]` | Runs one step: check deps → implement → run gates → update state → one commit per step. No arg = next pending. |
| `/plan-status` | Read-only progress view. |

> A project-local command of the same name shadows the plugin's; invoke explicitly with `/dev-loop:step` if needed.

## Reference

- Protocol, `dev-state.json` JSON schema, and the *implement-vs-gate* framework: [`skills/dev-loop/SKILL.md`](./skills/dev-loop/SKILL.md)
- Templates: [`skills/dev-loop/assets/`](./skills/dev-loop/assets/)

For install instructions and the full write-up, see the [repository README](../../README.md).
