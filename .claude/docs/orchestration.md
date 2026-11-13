# Orchestration: Claude and Codex through Orca

> Added by ccgs-orchestrate. Not part of upstream CCGS.

CCGS runs on Claude Code. This fork lets a Claude Code session, the
**coordinator**, hand bounded tasks to **Orca workers** (Codex or Claude) and
get the results back. Orca's `orchestration` skill does the dispatching; this
page covers using it on a CCGS project.

## Prerequisites

- Orca, with its `orchestration` and `orca-cli` skills installed
  (`orca skills install`). Do not copy those skills into this repository: they
  load a guide matched to the installed Orca version.
- Codex CLI, signed in.
- Everything CCGS itself needs (`.claude/docs/setup-requirements.md`).
- The project is a git repository, registered in Orca as kind `git`
  (`orca repo add --path <path>`). `--worktree new-child` needs git.

## Who does what

| Role | Runs | Why |
|---|---|---|
| Coordinator (your Claude Code session) | CCGS skills (`/prototype`, `/create-stories`, `/story-readiness`, `/code-review`, `/story-done`, …) and every commit | The skills need Claude Code tools; commits go through `validate-commit.sh` |
| Codex worker | A bounded task: implement one story or change, write tests, review a diff | A second model; it follows `AGENTS.md` |
| Claude worker | The same kinds of task, or a CCGS skill in a separate worktree | Gets CCGS hooks and rules in that worktree |

Typical loops:

- **Implement, then cross-review.** The coordinator gets a story ready
  (`/story-readiness`), a Codex worker implements it, and the coordinator runs
  `/code-review` and `/story-done`, then commits.
- **Second opinion.** Claude implements; a Codex worker reviews the diff
  without editing and reports findings.
- **Parallel features.** One worker per child worktree, on paths that do not
  overlap.

## Dispatching

From the coordinator, ask in plain words ("have Codex implement story X and
report back"); the orchestration skill builds the command. Underneath it is:

```text
orca orchestration worker-start --agent codex --worktree current --spec-file <spec.md>
```

- `--worktree current`: the coordinator's own worktree. The default for a single
  writer: the coordinator sees the diff directly and commits it.
- `--worktree new-child --name <short-name>`: an isolated worktree on its own
  branch, for parallel work. See [Commits](#commits).
- `--model <id>` and `--effort <level>`: choose a model per worker. This fork
  leaves the CCGS skill `model:` pins alone; worker models are chosen here.

Never run two writers on overlapping paths in the same worktree.

## Task spec template

Orca expects each spec to be self-contained. For CCGS work, include:

```markdown
## Task
<one sentence: the change and why>

## Context
- Story or design: <path, and the sections to read, by heading>
- Engine: <engine.name from project.yaml>; code root: <path>

## May edit
- <paths or globs>. Nothing else; use `ask` if you need more.

## Must not
- commit, push, tag or merge
- edit `.claude/`, `project.yaml` or design docs unless listed above
- use AskUserQuestion; send questions to the coordinator with `ask` (Claude workers)

## Done when
- <tests to run and the expected result>
- <for player-visible changes: the screenshot path under production/qa/evidence/>
- Report the files changed, the commands run with their exit codes, and open questions.
```

## Claude workers and the ask-first protocol

A Claude worker loads `CLAUDE.md`, so it follows `modes.automation`. At the
default, `collaborative`, it asks before every write, and a question shown in
the worker's own Orca tab stalls it until someone answers there. Either:

- give it a spec whose *May edit* list is the approval, and tell it to route
  questions through `ask` (the template above does both); or
- for runs you supervise, set `modes.automation: guided` or `autonomous` in
  `project.local.yaml`. `autonomous` logs each decision to
  `production/session-logs/decision-log.md`; `hotfix`, `gate-check`,
  `day-one-patch` and `setup-engine` ask anyway.

`project.local.yaml` and `.claude/settings.local.json` are gitignored, so a
`new-child` worktree may not have them. Claude Code copies the files listed in
`.worktreeinclude` into the worktrees it creates. Whether Orca does the same is
not verified yet; check before you rely on it.

## Commits

Codex never commits (`AGENTS.md`). Commits come from a Claude Code session whose
project is the worktree being committed, so `validate-commit.sh` checks them.

- **Worker in `current`:** review `git diff`, then commit from the coordinator
  as usual.
- **Worker in a child worktree:** start a Claude worker in that worktree to
  review and commit. Do not run `git -C <child> commit` from the coordinator:
  `validate-commit.sh` skips its checks when `-C` names another repository, and
  a worktree's top level counts as another one.

## Known gaps

- **Hooks in Codex.** Codex has its own hook system, and the CCGS hooks are not
  wired into it. `AGENTS.md` asks Codex to apply them by hand.
- **Orca repository identity.** When this fork was registered, Orca took its
  identity from the `upstream` remote (the original CCGS), not from `origin`.
  Check the target before creating a pull request from Orca.
- **Skill `model:` pins** apply when a skill is typed as `/name`
  (`.claude/docs/model-tiers.md`); worker models are set per dispatch.
