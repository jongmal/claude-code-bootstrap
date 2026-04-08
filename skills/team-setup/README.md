# 🤝 /team-setup — 팀 에이전트 오케스트레이션

> Claude Code에서 PM(Lead) + 팀메이트(BE/FE/QA 등) 구조를 한 번에 셋업하는 스킬

## 🤔 어떤 문제를 해결하나?

Claude Code에서 여러 역할(백엔드, 프론트엔드, QA)을 협업시키려면 `TeamCreate` → `Agent` spawn → `SendMessage` 통신을 직접 구성해야 합니다. 환경마다 tmux 유무, OS 차이, 의존성 누락 등 변수가 많아서 매번 삽질하게 됩니다.

이 스킬은 환경을 자동 감지하고, 누락된 의존성을 해결하고, 최적의 모드로 팀을 구성해줍니다.

## 🌍 지원 환경

| 환경 | 모드 | 비고 |
|------|------|------|
| 🐧 WSL + tmux | tmux pane | 권장! 팀메이트를 별도 pane에서 실시간 확인 |
| 🍎 macOS + tmux | tmux pane | 동일 |
| 🪟 Windows native | in-process | tmux 없이 Shift+Down으로 전환 |
| 🚫 tmux 미설치 | in-process | 설치 안내 → 선택 → 폴백 |

## 📥 설치

```bash
# 글로벌 (모든 프로젝트에서 사용)
cp -r team-setup ~/.claude/skills/

# 또는 프로젝트별
cp -r team-setup <프로젝트>/.claude/skills/
```

## ⚙️ 사전 설정

`~/.claude/settings.json`에 Agent Teams 활성화:
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

## 🎬 사용법

Claude Code에서 `/team-setup` 입력하면 스킬이 알아서 진행합니다:

1. 🔍 **환경 감지** — OS, tmux, Agent Teams 설정 자동 체크
2. 📦 **의존성 해결** — 없으면 설치 안내 or in-process 폴백
3. 💬 **프로젝트 분석** — 팀 이름, 역할, 작업 경로 질문
4. 🌳 **워크트리 준비** — 모노레포면 git worktree 자동 생성
5. 🖥️ **tmux 레이아웃** — PM | BE / FE 등 pane 구성
6. 🚀 **팀 spawn** — `TeamCreate` → `Agent` 병렬 생성

## ✅ 셋업 결과

```
Team: my-project
├── 👑 team-lead (PM) ← 현재 세션
├── 🔧 backend@my-project (BE, tmux pane)
└── 🎨 frontend@my-project (FE, tmux pane)
```

셋업 후:
```
# 팀메이트에게 지시
SendMessage(to: "backend", message: "GET /api/health 엔드포인트 추가해주세요")

# 작업 생성/관리
TaskCreate(title: "Health API", owner: "backend")
TaskList()
```

## 🔄 세션 재시작 시

팀메이트는 일회성이라 `--resume`으로 복원이 안 됩니다.
재시작 후 `/team-setup` 다시 실행하면 됩니다.

## 🔧 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| "Agent type not found" | 세션 중 에이전트 파일 생성 | Claude Code 재시작 |
| Git LFS hook 에러 | `git-lfs` 미설치 | `.git/hooks/post-checkout` 비활성화 |
| WSL 워크트리 경로 에러 | `.git`이 Windows 경로 참조 | 워크트리 아카이빙 후 재생성 |
| 팀메이트가 팀 소속 아님 | `team_name` 누락 | `TeamCreate` 먼저 실행 확인 |
| tmux 세션 밖에서 실행 | tmux 미시작 | `tmux new -s claude` 후 재실행 |
