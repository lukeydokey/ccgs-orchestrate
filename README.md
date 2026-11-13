# ccgs-orchestrate

**English** | [한국어](README-KR.md)

[Claude Code Game Studios](https://github.com/Donchitos/Claude-Code-Game-Studios)
(CCGS), with Codex and Claude working side by side. A Claude Code session runs
the CCGS skills and makes every commit; Orca workers (Codex or Claude) take
bounded tasks and report back.

This is an unofficial fork, based on upstream **v1.3.0**. For what CCGS itself
does (its skills, agents and hooks, and how a game grows), read the
[upstream README](https://github.com/Donchitos/Claude-Code-Game-Studios#readme).

## How it fits together

```text
Donchitos/Claude-Code-Game-Studios  <- upstream template
   | merged into
   v
ccgs-orchestrate (this repo)  <- adds AGENTS.md and the orchestration guide
   | cloned into; updates pulled from
   v
your game repo  <- where you work
   |- Claude Code (coordinator): CCGS skills, every commit
   `- Orca workers: Codex or Claude, one task each
```

| Added by this fork | What it is for |
|---|---|
| [`AGENTS.md`](AGENTS.md) | What Codex reads: the CCGS rules it must apply by hand, and that it never commits |
| [`.claude/docs/orchestration.md`](.claude/docs/orchestration.md) | Who does what, how to dispatch a worker, the task spec template, the commit path |
| `CLAUDE.md` (one line) | Points Claude Code to the orchestration guide |

## 1. Install the tools

Commands are for Windows PowerShell; the `git`, `gh` and `orca` commands are the
same in bash.

| Tool | Why | Install |
|---|---|---|
| Git for Windows | Version control. Its Git Bash runs every CCGS hook | [git-scm.com/downloads/win](https://git-scm.com/downloads/win) |
| Python 3 | Every skill and hook reads `project.yaml` through it | [python.org](https://www.python.org/) |
| Claude Code | The coordinator | `irm https://claude.ai/install.ps1 \| iex` |
| Codex CLI | The second model | `npm install -g @openai/codex` |
| Orca | Runs the workers | [orca.dev/download](https://orca.dev/download), then `orca skills install --skill orchestration --skill orca-cli` |
| GitHub CLI | Creating your game's repository (optional) | `winget install GitHub.cli` |
| jq | Sturdier JSON parsing in hooks (recommended) | `winget install jqlang.jq` |
| Chrome or Edge | `/prototype` plays and screenshots each build | — |
| Node.js | Growing a game into a web project | [nodejs.org](https://nodejs.org/) |

Run `claude` and `codex` once each to sign in, then check:

```powershell
claude --version
codex --version
python --version   # Python 3.x
orca status        # runtimeState: ready
```

## 2. Start a game project

```powershell
Set-Location C:\Dev
git clone https://github.com/lukeydokey/ccgs-orchestrate.git my-game
Set-Location C:\Dev\my-game
git remote rename origin ccgs   # framework updates come from here (step 8)
```

To keep the game on GitHub, give it its own repository. `--private` keeps it to
you; use `--public` to share it.

```powershell
# If the repository is or may become public, commit with your GitHub noreply address
$u = gh api user | ConvertFrom-Json
git config user.email "$($u.id)+$($u.login)@users.noreply.github.com"

gh repo create my-game --private --source . --remote origin --push
```

## 3. Register it in Orca

```powershell
orca repo add --path C:\Dev\my-game
```

Open the project in Orca and start Claude Code **at the project root**, the
folder that holds `CLAUDE.md`. Started from a subfolder, the CCGS hooks and
permission rules do not load.

## 4. Choose how often Claude asks (optional)

CCGS asks before every write by default. To change that for yourself only, run
this in Claude Code:

```text
/project-settings --local modes.automation=guided
```

| Value | Behaviour |
|---|---|
| `collaborative` (default) | Asks before every decision and every write |
| `guided` | Asks only for major decisions; proceeds on minor ones |
| `autonomous` | Decides, proceeds and logs each decision to `production/session-logs/decision-log.md` |

The setting goes to `project.local.yaml`, which is gitignored. Personal
permission rules go in `.claude/settings.local.json`; see
[`.claude/docs/settings-local-template.md`](.claude/docs/settings-local-template.md).

## 5. Make the game

In Claude Code:

| Command | When |
|---|---|
| `/prototype <idea>` | Describe a game; get a one-page brief and a playable browser build |
| `/start` | Guided onboarding, when you are not sure where to begin |
| `/next-step` | Names the one next step, wherever you are |
| `/create-stories`, `/dev-story`, `/story-done` | Grow the game one story at a time |
| `/code-review` | Review a change before committing it |

Every command is listed in the
[upstream README](https://github.com/Donchitos/Claude-Code-Game-Studios#slash-commands).

## 6. Bring in Codex

Ask the coordinator in plain words. Orca's `orchestration` skill starts the
worker, and the coordinator is woken when the worker reports.

```text
Have Codex implement story <id> in this worktree and report back.
Have Codex review the current diff without editing anything and report its findings.
Run two workers in child worktrees: Codex on <task A>, Claude on <task B>.
Use Codex with model <id> for this one.
```

- Codex follows [`AGENTS.md`](AGENTS.md): it edits only the paths its task names
  and never commits.
- One writer per worktree at a time; parallel work goes in child worktrees.
- The task spec template and the details are in
  [`.claude/docs/orchestration.md`](.claude/docs/orchestration.md).

## 7. Commit

Commit from Claude Code, in the worktree the change is in, so CCGS's
`validate-commit.sh` checks it. Use Conventional Commits (`feat:`, `fix:`,
`docs:` …).

- **Work in your own worktree:** review `git diff`, then ask Claude to commit.
- **Work in a child worktree:** have a Claude worker in that worktree review and
  commit it. A `git -C <child> commit` from the coordinator skips the CCGS checks.

## 8. Get framework updates

```powershell
Set-Location C:\Dev\my-game
git fetch ccgs
git merge ccgs/main
```

Conflicts usually come from files you have filled in, such as `CLAUDE.md`,
`project.yaml`, and `README.md` once it describes your game: keep your content
and take the structural changes. Notes for each version are in
[`UPGRADING.md`](UPGRADING.md).

<!-- docs/migration-guide-v1.1.md links to #configuration: keep this heading. -->
## Configuration

`project.yaml` holds the engine, the modes and the goal; `/project-settings`
shows and changes it. What each setting does:
[upstream README, Configuration](https://github.com/Donchitos/Claude-Code-Game-Studios#configuration)
and [`.claude/docs/effects-map.md`](.claude/docs/effects-map.md).

## Maintaining this fork

For the owner of this repository, to bring in an upstream release:

```powershell
Set-Location C:\Dev\ccgs-orchestrate
git fetch --tags upstream main
git merge upstream/main
```

- `README.md` belongs to this fork. On a conflict keep ours
  (`git checkout --ours README.md`, then `git add README.md`), then read what
  upstream changed for anything worth carrying over:
  `git log -p ORIG_HEAD..upstream/main -- README.md`.
- Update "based on upstream vX.Y.Z" in both `README.md` and `README-KR.md`.
- Pushing to `upstream` is disabled (`git remote set-url --push upstream no_push`);
  push to `origin`.

## Known gaps

- CCGS hooks do not run inside Codex; `AGENTS.md` asks Codex to apply them by hand.
- Orca may take a repository's identity from a remote other than `origin` (it
  chose `upstream` for this fork). Check the target before creating a pull
  request from Orca.
- Whether Orca copies gitignored local settings (`project.local.yaml`,
  `.claude/settings.local.json`) into child worktrees is not verified yet.

Details: [`.claude/docs/orchestration.md`](.claude/docs/orchestration.md#known-gaps).

## Credits and license

CCGS is by [Donchitos](https://github.com/Donchitos), under the MIT License; this
fork keeps that license ([`LICENSE`](LICENSE)). To support CCGS's author, see
[Supporting This Project](https://github.com/Donchitos/Claude-Code-Game-Studios#supporting-this-project)
in the upstream README.

Problems with this fork's additions: open an issue in this repository. Security
reports: [`SECURITY.md`](SECURITY.md).
