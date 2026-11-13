# AGENTS.md — ccgs-orchestrate

Instructions for Codex and other agents that read `AGENTS.md`. Claude Code
reads `CLAUDE.md` instead; nothing here changes how Claude Code behaves.

This repository is **ccgs-orchestrate**, an unofficial fork of
[Claude Code Game Studios](https://github.com/Donchitos/Claude-Code-Game-Studios)
(CCGS). CCGS is built for Claude Code: its skills, agents and hooks live under
`.claude/`. This fork adds a way to bring Codex in as an Orca worker next to
Claude. `.claude/docs/orchestration.md` describes who does what.

## Read first

`CLAUDE.md` is the master configuration. Its `@path` lines are Claude Code
imports that you do not expand, so read those files yourself:

- `.claude/docs/directory-structure.md`: where things live. The code root
  depends on `engine.name`; resolve it before you write any source file.
- `.claude/docs/coordination-rules.md`
- `.claude/docs/coding-standards.md`, including the test-evidence table that
  decides when a change counts as done.

Then:

- `project.yaml`, and `project.local.yaml` if it exists (it overrides
  `project.yaml`): engine, modes and goal. Precedence is in
  `.claude/docs/config-resolution.md`.
- `docs/engine-reference/<engine>/VERSION.md` once an engine is set: the pinned
  engine version. Do not assume APIs from any other version.
- `production/session-state/active.md` if it exists: the current session
  checkpoint.

## Path-scoped rules

Each file in `.claude/rules/` starts with a `paths:` list of globs. Claude Code
loads a rule when it touches a matching file; you do not get that for free.
Before you edit a file, read every rule whose `paths:` matches it and follow it.

## Hooks do not run for you

`.claude/settings.json` wires Claude Code hooks that check reads, writes, data
files and commits. None of them run in Codex, so apply them yourself:

- **Do not commit, push, tag or merge.** Commits are made from a Claude Code
  session so that `validate-commit.sh` checks them. Leave your changes in the
  working tree and report them.
- Keep data JSON (`assets/data/**`, `data/**`) valid.
- Read large design docs (GDDs, ADRs, the art bible) by section: search for
  `^#` to find the headings, then read only the range you need.
- Keep to the deny list in `.claude/settings.json`: no `rm -rf`, no force push,
  no `git reset --hard`, no `git clean -f`, and never read or edit `.env*`.

## Skills

`.claude/skills/<name>/SKILL.md` are Claude Code skills. They call tools you do
not have (`AskUserQuestion`, `Agent`, `TaskCreate`), so do not try to run one
end to end. When a task names a skill, read its `SKILL.md` as a procedure and:

- send a question meant for the user to whoever dispatched you (Orca `ask`),
  or to the user when you are not a worker;
- do a subagent step yourself, or report it as not done;
- write only files that both the procedure and your task name.

## Asking before acting

CCGS asks before it writes by default (`modes.automation`, described in
`.claude/docs/effects-map.md`).

- **As an Orca worker** (your prompt carries Task and Dispatch IDs): the task
  spec is the approval. Edit only the paths it names. Anything outside them,
  and any design decision the spec leaves open, goes to your coordinator with
  `ask`.
- **Working with the user directly:** follow `modes.automation` the way Claude
  would.

## Never

- Open, comment on or push to an issue or pull request in the upstream CCGS
  repository or in any engine, plugin or vendor repository. Draft it and let
  the user submit it.
- Add any of the six knobs that `modes.rigor` fronts to `project.yaml` (the
  comment at the top of `project.yaml` lists them).
- Edit framework files under `.claude/` unless the task is framework work.

## Done means evidence

Report what proves the change works: the tests you ran and their result, and
for anything a player can see, the screenshot path under
`production/qa/evidence/` (`.claude/docs/run-and-observe.md`). A parse check is
not a run.
