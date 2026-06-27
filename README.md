# dev-loop

A **plan + state + `/step`** loop-engineering harness for [Claude Code](https://docs.claude.com/en/docs/claude-code/overview). Drive any project through a verified, resumable, one-step-at-a-time plan. State lives in files — so you can `/clear` between steps and pick up exactly where you left off.

> This repository is both a Claude Code **plugin** (`dev-loop`) and a single-plugin **marketplace** that hosts it.

## What it gives you

Two files become the source of truth — not the chat history:

- **`.claude/dev-plan.md`** — the spec (*what to do*): an ordered list of steps, each with a goal, success criteria, verification method, and dependencies.
- **`.claude/dev-state.json`** — the ledger (*what happened* + config): per-step status, gate results, learnings, and **all settings** (working branch, quality-gate commands, project name, plan path).

Because every setting lives in `dev-state.json`, the commands are completely project-agnostic. The same `/step` works in any repository.

## Commands

| Command | What it does |
|---|---|
| `/plan-init [goal · spec · path to a review doc]` | Scaffolds `dev-plan.md` + `dev-state.json`. Auto-detects quality gates (typecheck / test / lint / build), confirms them with you, records a baseline (HEAD commit + gate results), and sets up the review gate. |
| `/step [id]` | Runs **one** step: check dependencies → implement → run the gates → **hierarchical review gate** → update state → **one commit per step**. With no argument it runs the next pending step (and resumes an in-progress one). |
| `/step-review [id]` | Read-only: runs **only** the review gate against the current working tree and reports the verdict. Never edits, commits, or advances `currentStep`. Useful after manual fixes or as a pre-commit dry run. |
| `/plan-status` | Read-only progress view: step table (with review verdicts), completion %, gate-regression check, blocked steps' human tasks, accumulated learnings. |

> If a project already defines a command with the same name locally (e.g. `.claude/commands/step.md`), the local one wins. In that case invoke the plugin version explicitly as `/dev-loop:step`.

## Hierarchical review gate

At the end of every `/step` (after the quality gates pass, before commit), dev-loop runs a **self-contained, hierarchical, iterative code review** — no external plugin required:

- **Layer 1 (breadth)** — one `dev-reviewer` sub-agent per perspective (correctness, architecture, security, testing, performance, … — a built-in catalog comparable in breadth to a full PR review) reviews the step's diff in parallel and returns labeled findings (`[must]/[ask]/[suggestion]/[nits]`).
- **Layer 2 (meta — *review of the review*)** — a `dev-review-meta` sub-agent dedupes, refutes false positives against the real code, classifies each finding by scope (in-step / out-of-scope / dependency), reconciles against the step's success criteria, and decides what truly must be fixed.
- **Iterate** — fixable `[must]` findings are auto-fixed, the gates re-run, and the diff is re-reviewed for several rounds until it converges. An oscillation guard and a `maxRounds` cap prevent non-terminating loops.
- **Decide the next action** — on a clean result the step is committed and `currentStep` advances; findings that need a human (infra/config/credentials/judgment calls) leave the step **`blocked`** with a human task list instead of being auto-edited; valid out-of-scope findings become new steps.

Everything is configured in `dev-state.json`'s `review` block (`enabled`, `maxRounds`, `blockOn`, `autoFix`, `perspectives`). Set `enabled: false` (or omit the block) to keep the classic behavior.

## Install

From GitHub:

```text
/plugin marketplace add YuheiNakasaka/dev-loop
/plugin install dev-loop@dev-loop-marketplace
```

From a local clone (for development):

```text
/plugin marketplace add /path/to/dev-loop
/plugin install dev-loop@dev-loop-marketplace
```

To enable the plugin across all your projects automatically, add it to `~/.claude/settings.json`:

```jsonc
{
  "extraKnownMarketplaces": {
    "dev-loop-marketplace": { "source": { "source": "github", "repo": "YuheiNakasaka/dev-loop" } }
  },
  "enabledPlugins": { "dev-loop@dev-loop-marketplace": true }
}
```

## Quick start

```text
/plan-init "Refactor the auth flow: fix the token-refresh bug and add tests"
#   → detects gates → confirms → writes .claude/dev-plan.md + .claude/dev-state.json

/step          # run step 1, then commit
/clear         # safe to clear — the state is on disk
/step          # run step 2 (auto-resumes from state)
...
/plan-status   # check progress any time
```

## Why externalize state? (loop engineering)

On long-horizon tasks the context window fills up; summarization or `/clear` loses the thread. By keeping the **plan** (spec) and **state** (ledger) in files, each `/step` reconstructs *what to do next* and *what already happened* from disk. The result:

- **Resumable** — survives `/clear`, compaction, and fresh sessions.
- **Verified** — every step must pass the project's quality gates, so regressions surface immediately.
- **Traceable** — one commit per step makes `git bisect` trivial, and `learnings` accumulate project-specific knowledge.

This follows the same lineage as [GitHub Spec Kit](https://github.com/github/spec-kit) (spec → plan → tasks) and Anthropic's writing on [effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — external memory plus a verification loop.

## State schema & design principles

The full execution protocol, the `dev-state.json` JSON schema, the `dev-plan.md` structure, the *implement-vs-gate* decision framework, and the **hierarchical review gate** protocol all live in the bundled skill — the single source of truth:

- [`plugins/dev-loop/skills/dev-loop/SKILL.md`](./plugins/dev-loop/skills/dev-loop/SKILL.md)
- Templates: [`plugins/dev-loop/skills/dev-loop/assets/`](./plugins/dev-loop/skills/dev-loop/assets/)

## Repository layout

```
dev-loop/
├── .claude-plugin/marketplace.json     # marketplace manifest
├── README.md
├── LICENSE
└── plugins/dev-loop/                   # the plugin
    ├── .claude-plugin/plugin.json
    ├── README.md
    ├── commands/                       # /step, /step-review, /plan-init, /plan-status
    ├── agents/                         # dev-reviewer, dev-review-meta (built-in review engine)
    └── skills/dev-loop/                # SKILL.md (protocol + schema) + assets/ (templates + review-perspectives)
```

> The command prose is written in Japanese — that is the workflow the author uses day to day. The commands work regardless of your Claude Code UI language; translation PRs are welcome.

## Contributing

Issues and pull requests are welcome. Good first contributions: English translations of the command prose, more gate-detection recipes (Cargo, Go, Python, Gradle…), and example plans.

## License

[MIT](./LICENSE) © Yuhei Nakasaka
