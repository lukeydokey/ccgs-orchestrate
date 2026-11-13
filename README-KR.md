# ccgs-orchestrate

[English](README.md) | **한국어**

Codex와 Claude가 함께 일하도록 만든
[Claude Code Game Studios](https://github.com/Donchitos/Claude-Code-Game-Studios)(CCGS)입니다.
Claude Code 세션이 CCGS 스킬을 실행하고 모든 커밋을 맡습니다. Orca 워커(Codex 또는
Claude)는 범위가 정해진 작업을 받아 결과를 보고합니다.

원본 **v1.3.0**을 기준으로 한 비공식 fork입니다. CCGS 자체의 기능(스킬, 에이전트, 훅,
게임을 키워 가는 방식)은 [원본 README](https://github.com/Donchitos/Claude-Code-Game-Studios#readme)를
참고하세요.

## 전체 구조

```text
Donchitos/Claude-Code-Game-Studios  <- 원본 템플릿
   | merge
   v
ccgs-orchestrate (이 저장소)  <- AGENTS.md와 오케스트레이션 안내 추가
   | clone, 업데이트 pull
   v
내 게임 저장소  <- 실제로 작업하는 곳
   |- Claude Code (코디네이터): CCGS 스킬, 모든 커밋
   `- Orca 워커: Codex 또는 Claude, 작업 하나씩
```

| 이 fork가 추가한 것 | 용도 |
|---|---|
| [`AGENTS.md`](AGENTS.md) | Codex가 읽는 파일. Codex가 직접 지켜야 하는 CCGS 규칙과 커밋 금지 원칙 |
| [`.claude/docs/orchestration.md`](.claude/docs/orchestration.md) | 역할 분담, 워커 실행 방법, 작업 지시 템플릿, 커밋 경로 |
| `CLAUDE.md` (한 줄) | Claude Code에 오케스트레이션 안내 문서의 위치를 알려줌 |

## 1. 도구 설치

명령은 Windows PowerShell 기준입니다. `git`, `gh`, `orca` 명령은 bash에서도 같습니다.

| 도구 | 용도 | 설치 |
|---|---|---|
| Git for Windows | 버전 관리. 함께 설치되는 Git Bash가 CCGS 훅을 실행 | [git-scm.com/downloads/win](https://git-scm.com/downloads/win) |
| Python 3 | 모든 스킬과 훅이 이걸로 `project.yaml`을 읽음 | [python.org](https://www.python.org/) |
| Claude Code | 코디네이터 | `irm https://claude.ai/install.ps1 \| iex` |
| Codex CLI | 두 번째 모델 | `npm install -g @openai/codex` |
| Orca | 워커 실행 | [orca.dev/download](https://orca.dev/download)에서 설치 후 `orca skills install --skill orchestration --skill orca-cli` |
| GitHub CLI | 게임 저장소 만들기 (선택) | `winget install GitHub.cli` |
| jq | 훅의 JSON 처리를 더 안정적으로 (권장) | `winget install jqlang.jq` |
| Chrome 또는 Edge | `/prototype`이 빌드를 실행하고 스크린샷을 찍음 | — |
| Node.js | 게임을 웹 프로젝트로 키울 때 | [nodejs.org](https://nodejs.org/) |

`claude`와 `codex`를 한 번씩 실행해서 로그인한 뒤 확인합니다.

```powershell
claude --version
codex --version
python --version   # Python 3.x
orca status        # runtimeState: ready
```

## 2. 게임 프로젝트 시작

```powershell
Set-Location C:\Dev
git clone https://github.com/lukeydokey/ccgs-orchestrate.git my-game
Set-Location C:\Dev\my-game
git remote rename origin ccgs   # 프레임워크 업데이트는 여기서 받음 (8단계)
```

게임을 GitHub에 올리려면 게임 전용 저장소를 만듭니다. `--private`는 나만 볼 수 있고,
공유하려면 `--public`을 씁니다.

```powershell
# 저장소가 공개이거나 나중에 공개할 수 있다면, 커밋 이메일을 GitHub noreply 주소로
$u = gh api user | ConvertFrom-Json
git config user.email "$($u.id)+$($u.login)@users.noreply.github.com"

gh repo create my-game --private --source . --remote origin --push
```

## 3. Orca에 등록

```powershell
orca repo add --path C:\Dev\my-game
```

Orca에서 프로젝트를 열고, **프로젝트 루트**(`CLAUDE.md`가 있는 폴더)에서 Claude Code를
시작합니다. 하위 폴더에서 시작하면 CCGS 훅과 권한 규칙이 적용되지 않습니다.

## 4. Claude가 묻는 빈도 정하기 (선택)

CCGS는 기본적으로 파일을 쓰기 전마다 묻습니다. 나에게만 적용되게 바꾸려면 Claude Code에서
실행합니다.

```text
/project-settings --local modes.automation=guided
```

| 값 | 동작 |
|---|---|
| `collaborative` (기본값) | 모든 결정과 모든 파일 쓰기 전에 물음 |
| `guided` | 중요한 결정만 묻고, 사소한 것은 바로 진행 |
| `autonomous` | 스스로 결정하고 진행하며, 결정마다 `production/session-logs/decision-log.md`에 기록 |

이 설정은 git에 올라가지 않는 `project.local.yaml`에 저장됩니다. 개인 권한 규칙은
`.claude/settings.local.json`에 넣습니다.
[`.claude/docs/settings-local-template.md`](.claude/docs/settings-local-template.md)를 참고하세요.
두 파일 모두 `.worktreeinclude`에 적혀 있어서, Orca와 Claude Code가 새 작업 폴더를 만들 때
함께 복사합니다.

## 5. 게임 만들기

Claude Code에서:

| 명령 | 언제 |
|---|---|
| `/prototype <아이디어>` | 게임을 설명하면 한 쪽짜리 기획서와 바로 해 볼 수 있는 브라우저 빌드를 만듦 |
| `/start` | 어디서 시작할지 모를 때 안내를 받으며 시작 |
| `/next-step` | 지금 상태에서 할 일 하나를 알려줌 |
| `/create-stories`, `/dev-story`, `/story-done` | 스토리 단위로 게임을 키움 |
| `/code-review` | 커밋 전에 변경 사항 리뷰 |

전체 명령 목록은 [원본 README](https://github.com/Donchitos/Claude-Code-Game-Studios#slash-commands)에
있습니다.

## 6. Codex 불러오기

코디네이터에게 평소 말투로 요청하면 됩니다. Orca의 `orchestration` 스킬이 워커를 띄우고,
워커가 보고하면 코디네이터가 깨어나 이어서 진행합니다.

```text
Codex한테 이 작업 폴더에서 스토리 <id>를 구현하고 결과 보고하라고 해줘.
Codex한테 지금 변경 사항을 수정 없이 리뷰만 하고 결과를 보고하라고 해줘.
별도 작업 폴더 두 개에서 Codex는 <작업 A>, Claude는 <작업 B>를 동시에 진행해줘.
이번 작업은 Codex를 <모델 id>로 돌려줘.
```

- Codex는 [`AGENTS.md`](AGENTS.md)를 따릅니다. 작업 지시에 적힌 경로만 수정하고, 커밋은
  하지 않습니다.
- 한 작업 폴더에서는 한 번에 한 워커만 파일을 수정합니다. 동시에 진행할 작업은 별도 작업
  폴더(child worktree)에서 합니다.
- 새 작업 폴더는 원격 기본 브랜치(`origin/main`)에서 만들어집니다. 아직 push하지 않은
  커밋이 있다면 먼저 push하거나, 내 브랜치에서 시작하도록 요청하세요(`--base-branch main`).
- 작업 지시 템플릿과 자세한 내용은
  [`.claude/docs/orchestration.md`](.claude/docs/orchestration.md)에 있습니다.

## 7. 커밋

변경 사항이 있는 작업 폴더의 Claude Code에서 커밋합니다. 그래야 CCGS의
`validate-commit.sh`가 검사합니다. 커밋 메시지는 Conventional Commits 형식(`feat:`,
`fix:`, `docs:` …)을 씁니다.

- **내 작업 폴더에서 한 작업:** `git diff`를 확인한 뒤 Claude에게 커밋을 요청합니다.
- **별도 작업 폴더에서 한 작업:** 그 작업 폴더에 Claude 워커를 띄워 리뷰하고 커밋하게
  합니다. 코디네이터에서 `git -C <작업 폴더> commit`으로 커밋하면 CCGS 검사를 건너뜁니다.

## 8. 프레임워크 업데이트 받기

```powershell
Set-Location C:\Dev\my-game
git fetch ccgs
git merge ccgs/main
```

충돌은 주로 직접 채운 파일에서 납니다. `CLAUDE.md`, `project.yaml`, 그리고 게임 설명으로
바꾼 뒤의 `README.md` 같은 파일입니다. 내 내용은 유지하고 구조 변경만 받아들이세요. 버전별
안내는 [`UPGRADING.md`](UPGRADING.md)에 있습니다.

## 설정

`project.yaml`에 엔진, 모드, 목표가 들어 있고, `/project-settings`로 보고 바꿀 수 있습니다.
설정별 동작은
[원본 README의 Configuration](https://github.com/Donchitos/Claude-Code-Game-Studios#configuration)과
[`.claude/docs/effects-map.md`](.claude/docs/effects-map.md)를 참고하세요.

## 이 fork 관리하기

이 저장소 소유자가 원본의 새 릴리스를 받는 방법입니다.

```powershell
Set-Location C:\Dev\ccgs-orchestrate
git fetch --tags upstream main
git merge upstream/main
```

- `README.md`는 이 fork 전용입니다. 충돌하면 내 것을 유지하고
  (`git checkout --ours README.md` 후 `git add README.md`), 원본에서 바뀐 내용 중 가져올
  것이 있는지 확인합니다: `git log -p ORIG_HEAD..upstream/main -- README.md`
- `README.md`와 `README-KR.md` 두 곳의 "원본 vX.Y.Z 기준" 문구를 갱신합니다.
- `upstream`으로는 push가 막혀 있습니다(`git remote set-url --push upstream no_push`).
  push는 `origin`으로 합니다.

## 알려진 한계

- Codex에서는 CCGS 훅이 실행되지 않습니다. 대신 `AGENTS.md`에서 Codex가 그 규칙을 직접
  지키도록 안내합니다.
- Orca가 저장소를 `origin`이 아닌 다른 원격 기준으로 인식할 수 있습니다. 이 fork는
  `upstream` 기준으로 인식됐습니다. Orca에서 PR을 만들 때는 대상 저장소를 확인하세요.
- CCGS 프로젝트에서 `/code-review`는 CCGS 스킬(명령 목록에 `(project)`로 표시)이 실행되고,
  Claude Code 기본 `/code-review`는 가려집니다. CCGS 파일 수십 개가 이 이름으로 참조하고
  있어서 이 fork는 이름을 바꾸지 않습니다.

자세한 내용: [`.claude/docs/orchestration.md`](.claude/docs/orchestration.md#known-gaps)

## 출처와 라이선스

CCGS는 [Donchitos](https://github.com/Donchitos)가 MIT 라이선스로 만든 프로젝트이고, 이
fork도 같은 라이선스를 따릅니다([`LICENSE`](LICENSE)). CCGS 원작자를 후원하려면 원본
README의 [Supporting This Project](https://github.com/Donchitos/Claude-Code-Game-Studios#supporting-this-project)를
참고하세요.

이 fork에서 추가한 부분의 문제는 이 저장소에 이슈로 남겨 주세요. 보안 문제는
[`SECURITY.md`](SECURITY.md)를 참고하세요.
