---
name: team-setup
description: |
  팀 에이전트 오케스트레이션 셋업.
  새 프로젝트에서 PM(Lead) + 팀메이트 구조를 자동 구성합니다.
  환경을 자동 감지하여 tmux/in-process 모드를 선택합니다.
  "/team-setup" 또는 "팀 셋업"이라고 말하면 활성화됩니다.
allowed-tools:
  - Read
  - Grep
  - Glob
  - Edit
  - Write
  - Bash
  - AskUserQuestion
  - TeamCreate
  - Agent
  - SendMessage
  - TaskCreate
  - TaskList
  - TaskUpdate
---

# Team Agent Orchestration Setup

Claude Code 팀 에이전트를 셋업하는 스킬.
환경을 자동 감지하여 최적의 구성을 선택한다.

---

## Step 0: 환경 감지 (가장 먼저 실행)

아래 명령을 실행하여 환경을 파악한다:

```bash
echo "=== OS ==="
uname -s  # Linux=WSL/Linux, Darwin=macOS, MINGW/MSYS=Windows Git Bash
cat /proc/version 2>/dev/null | grep -i microsoft && echo "WSL=true" || echo "WSL=false"

echo "=== Shell ==="
echo $SHELL
echo $TERM_PROGRAM  # vscode, iTerm2, etc.

echo "=== tmux ==="
tmux -V 2>/dev/null || echo "tmux=NOT_INSTALLED"
echo "TMUX=$TMUX"  # 비어있으면 tmux 세션 밖

echo "=== Claude Code ==="
claude --version 2>/dev/null || echo "claude=NOT_INSTALLED"

echo "=== Agent Teams ==="
grep -o 'AGENT_TEAMS.*' ~/.claude/settings.json 2>/dev/null || echo "AGENT_TEAMS=NOT_SET"

echo "=== Git ==="
git --version 2>/dev/null || echo "git=NOT_INSTALLED"
```

### 환경별 분기

| 감지 결과 | 모드 | 조치 |
|----------|------|------|
| WSL + tmux 설치 + tmux 세션 안 | **tmux 모드** | tmux 자동 구성 |
| WSL + tmux 미설치 | **설치 후 tmux** | tmux 설치 가이드 → 설치 → tmux 모드 |
| macOS + tmux 설치 | **tmux 모드** | tmux 자동 구성 |
| macOS + tmux 미설치 | **설치 후 tmux** | `brew install tmux` → tmux 모드 |
| Windows (native, Git Bash) | **in-process 모드** | tmux 없이 팀 에이전트 운영 |
| AGENT_TEAMS 미설정 | **설정 필요** | settings.json 자동 수정 |
| Claude Code 미설치 | **중단** | 설치 안내 후 종료 |

---

## Step 0-1: 누락 의존성 자동 해결

### Agent Teams 미활성화 시

```bash
# settings.json 읽어서 env 블록에 추가
# 파일이 없으면 새로 생성
```

settings.json에 아래 내용이 없으면 자동 추가:
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

수정 후 사용자에게 알림: "Agent Teams를 활성화했습니다. Claude Code를 재시작해야 적용됩니다."
→ `AskUserQuestion`으로 재시작 여부 확인.

### tmux 미설치 시 (Linux/WSL)

사용자에게 선택지 제안:

```
tmux가 설치되지 않았습니다. 두 가지 옵션이 있습니다:

A) tmux 설치 (권장 — 팀메이트를 별도 pane에서 시각적으로 확인 가능)
   → sudo apt install tmux (Ubuntu/Debian)
   → sudo yum install tmux (RHEL/CentOS)

B) in-process 모드로 진행 (tmux 없이 운영)
   → 팀메이트가 백그라운드에서 동작
   → SendMessage로만 통신 (시각적 pane 없음)
   → Shift+Down으로 팀원 간 전환

어떤 옵션을 선택하시겠습니까?
```

### tmux 미설치 시 (macOS)

```
tmux가 설치되지 않았습니다.

A) brew install tmux (Homebrew 필요)
B) in-process 모드로 진행
```

### Windows Native 환경

```
Windows 환경에서는 tmux를 사용할 수 없습니다.
in-process 모드로 팀 에이전트를 구성합니다.

팀메이트는 백그라운드에서 동작하며:
- SendMessage로 지시/보고
- Shift+Down으로 팀원 간 전환
- TaskList로 작업 현황 관리
```

---

## Step 1: 프로젝트 분석

사용자에게 질문:

1. **팀 이름**: 프로젝트/팀 식별자 (예: "alp", "erp")
2. **팀 구성**: 어떤 역할의 팀메이트가 필요한가?
   - 모노레포 (하나의 repo, 여러 앱) → 워크트리로 분리
   - 멀티레포 (여러 repo) → 각 repo 디렉토리 지정
   - 단일 repo → 같은 디렉토리, 역할만 분리
3. **작업 디렉토리**: 각 팀메이트의 작업 경로
4. **에이전트 타입**: `.claude/agents/` 에 커스텀 에이전트 정의가 있는가?

---

## Step 2: 워크트리 준비 (모노레포인 경우)

```bash
# Git LFS hook 비활성화 (worktree 생성 시 exit 2 방지)
if [ -f .git/hooks/post-checkout ]; then
  mv .git/hooks/post-checkout .git/hooks/post-checkout.disabled
fi

# 팀메이트별 워크트리 생성
git worktree add <경로> <브랜치>

# 예시:
git worktree add ../project-be feature/backend-work
git worktree add ../project-fe feature/frontend-work
```

**주의:**
- WSL에서 Windows 파일시스템(`/mnt/c/...`)의 기존 워크트리는 `.git` 파일 경로가 Windows 형식일 수 있음
- 깨진 워크트리는 아카이빙 후 새로 생성: `mv old-wt archive-wt-YYYYMMDD`

---

## Step 3: tmux 레이아웃 생성 (tmux 모드만)

**in-process 모드면 이 단계 건너뛴다.**

```bash
# 2팀원 (PM | BE / FE):
tmux split-window -h -c <BE경로>
tmux split-window -v -t :.1 -c <FE경로>
tmux select-layout main-vertical
tmux select-pane -t :.0

# 3팀원 (PM | BE / FE / QA):
tmux split-window -h -c <BE경로>
tmux split-window -v -t :.1 -c <FE경로>
tmux split-window -v -t :.2 -c <QA경로>
tmux select-layout main-vertical
tmux select-pane -t :.0

# pane 이름 설정
tmux select-pane -t :.0 -T "PM"
tmux select-pane -t :.1 -T "BE"
tmux select-pane -t :.2 -T "FE"
```

---

## Step 4: 팀 생성 + 팀메이트 spawn

**tmux 모드와 in-process 모드 공통.**

```
# 1. 팀 생성
TeamCreate(team_name: "<팀이름>", agent_type: "orchestrator")

# 2. 팀메이트 spawn (병렬, 백그라운드)
Agent(
  name: "<역할명>",
  team_name: "<팀이름>",
  mode: "bypassPermissions",
  subagent_type: "<에이전트타입>",  # .claude/agents/ 에 정의된 타입, 없으면 생략
  run_in_background: true,
  prompt: "당신은 <프로젝트>의 <역할> 담당 팀메이트입니다.
    먼저 `cd <작업경로>` 명령을 실행해서 작업 디렉토리를 변경하세요.
    기술 스택: <스택>
    CLAUDE.md를 읽고 코드베이스 상태를 파악한 뒤 팀 리드에게 상태를 보고해주세요."
)
```

**핵심 파라미터:**
- `name`: 팀메이트 이름 (SendMessage에서 `to` 로 사용)
- `team_name`: TeamCreate에서 만든 팀 이름
- `mode: "bypassPermissions"`: 팀메이트가 자유롭게 도구 사용
- `run_in_background: true`: 병렬 spawn
- `subagent_type`: 커스텀 에이전트 정의 (선택)

### tmux 모드 vs in-process 모드 차이

| | tmux 모드 | in-process 모드 |
|--|----------|----------------|
| 시각적 확인 | 별도 pane에서 실시간 확인 | 보이지 않음 |
| 팀원 전환 | tmux pane 전환 | Shift+Down |
| 통신 | SendMessage + pane 직접 확인 | SendMessage만 |
| 요구사항 | tmux + tmux 세션 안에서 실행 | 없음 |

---

## Step 5: 검증

```bash
# tmux 모드: pane 상태 확인
tmux list-panes -F '#{pane_index}: #{pane_title} #{pane_current_command}'

# 공통: 팀 설정 확인
cat ~/.claude/teams/<팀이름>/config.json
```

팀메이트가 자동으로 상태 보고 메시지를 보냄 → idle_notification 수신 확인.

---

## 팀 운영

### 작업 지시 (PM → 팀메이트)
```
SendMessage(to: "backend", message: "GET /api/v1/health 엔드포인트를 추가해주세요.")
```

### 작업 관리 (TaskList)
```
TaskCreate(title: "Health API 구현", description: "...", owner: "backend")
TaskList()
TaskUpdate(id: "task-1", status: "completed")
```

### 종료
```
SendMessage(to: "*", message: {type: "shutdown_request"})
TeamDelete(team_name: "<팀이름>")
```

---

## 세션 재시작 시 (resume)

팀메이트는 일회성 — resume으로 복원 불가. 매번 새로 생성:

```
1. claude --resume  (PM 오케스트레이터 복원)
2. TeamCreate → Agent spawn (팀메이트 재생성)
3. 팀메이트에게 "git status 확인 후 상태 보고" 지시
```

---

## 트러블슈팅

### "Agent type not found"
- `.claude/agents/<name>.md` 파일이 세션 시작 시점에 존재해야 함
- 세션 중 생성한 에이전트는 재시작 후 인식됨
- 또는 `subagent_type` 생략하고 prompt에 역할 직접 기술

### Git LFS hook 에러
```
"git-lfs was not found on your path"
```
→ `.git/hooks/post-checkout` 를 `.git/hooks/post-checkout.disabled` 로 이름 변경

### WSL 워크트리 경로 문제
```
"unable to locate repository; .git file does not reference a repository"
```
→ 워크트리의 `.git` 파일이 Windows 경로(`C:/Users/...`)를 참조
→ 아카이빙 후 WSL 경로로 새 워크트리 생성

### 팀메이트가 독립 세션으로 동작
- `team_name` 파라미터 누락 → 팀 소속 아닌 독립 에이전트가 됨
- 반드시 `TeamCreate` 먼저 실행 후 `Agent(team_name: ...)` 으로 spawn

### tmux pane에서 Claude가 안 보임
- 팀메이트는 `backendType: "tmux"` 로 자동 tmux pane 할당
- 수동으로 pane에 `claude` 실행하는 것은 팀 오케스트레이션이 **아님**
- TeamCreate → Agent spawn이 자동으로 pane 할당

### tmux 세션 밖에서 실행한 경우
```
"not running inside tmux"
```
→ `tmux new-session -s claude` 로 세션 생성 후 재시작
→ 또는 in-process 모드로 전환

### Windows에서 tmux 관련 에러
→ Windows native에서는 tmux 불가. in-process 모드 자동 선택됨.
→ WSL 사용을 권장: `wsl --install` 후 WSL 내에서 실행

---

## 프로젝트별 커스텀 에이전트 정의

`.claude/agents/team-<역할>.md` 파일로 팀메이트의 행동을 정의:

```markdown
---
name: team-<역할>
description: |
  <역할> 팀원 에이전트. <기술스택> 구현.
  PM(Lead)이 spawn하여 작업을 위임합니다.
tools:
  - Read
  - Grep
  - Glob
  - Edit
  - Write
  - Bash
---

# <역할> Team Agent

역할 정의, 행동 원칙, 금지 사항, 참조 문서, 보고 형식 등 기술.
```

에이전트 정의가 없으면 `subagent_type` 생략하고 prompt에 모든 지침을 포함.

---

## 설정 요구사항

### ~/.claude/settings.json (필수)
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### .claude/settings.local.json (프로젝트별, 선택)
```json
{
  "permissions": {
    "allow": [
      "Bash(tmux:*)",
      "Bash(git:*)",
      "Bash(pnpm:*)"
    ]
  }
}
```
