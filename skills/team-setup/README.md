# /team-setup — 팀 에이전트 오케스트레이션

Claude Code에서 PM(Lead) + 팀메이트(BE/FE/QA 등) 구조를 한 번에 셋업하는 스킬.

## 어떤 문제를 해결하나?

Claude Code에서 여러 역할(백엔드, 프론트엔드, QA)을 협업시키려면 `TeamCreate` → `Agent` spawn → `SendMessage` 통신을 직접 구성해야 한다. 환경마다 tmux 유무, OS 차이, 의존성 누락 등 변수가 많아 매번 삽질하게 된다.

이 스킬은 환경을 자동 감지하고, 누락된 의존성을 해결하고, 최적의 모드로 팀을 구성해준다.

## 지원 환경

| 환경 | 모드 | 비고 |
|------|------|------|
| WSL + tmux | tmux pane | 권장. 팀메이트를 시각적으로 확인 가능 |
| macOS + tmux | tmux pane | 동일 |
| Windows native | in-process | tmux 없이 Shift+Down으로 전환 |
| tmux 미설치 | in-process | 설치 안내 또는 폴백 |

## 사용법

### 1. 스킬 복사

```bash
# 글로벌 (모든 프로젝트에서 사용)
cp -r team-setup ~/.claude/skills/

# 프로젝트별
cp -r team-setup <프로젝트>/.claude/skills/
```

### 2. Agent Teams 활성화

`~/.claude/settings.json`:
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### 3. 실행

Claude Code에서:
```
/team-setup
```

스킬이 순서대로 안내한다:
1. 환경 감지 (OS, tmux, Agent Teams 설정)
2. 누락 의존성 해결 (자동 설치 또는 안내)
3. 프로젝트 구조 질문 (팀 이름, 역할, 작업 경로)
4. 워크트리 준비 (모노레포인 경우)
5. tmux 레이아웃 생성 (해당 시)
6. `TeamCreate` → `Agent` spawn

## 셋업 결과 예시

```
Team: my-project
├── team-lead (PM) ← 현재 세션
├── backend@my-project (BE, tmux pane)
└── frontend@my-project (FE, tmux pane)
```

셋업 후 사용:
```
# 팀메이트에게 지시
SendMessage(to: "backend", message: "GET /api/health 엔드포인트 추가해주세요")

# 작업 생성/관리
TaskCreate(title: "Health API", owner: "backend")
TaskList()
```

## 세션 재시작 시

팀메이트는 일회성이라 `--resume`으로 복원 안 됨. 재시작 후 `/team-setup` 다시 실행.

## 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| "Agent type not found" | 세션 시작 후 에이전트 파일 생성 | Claude Code 재시작 |
| Git LFS hook 에러 | `git-lfs` 미설치 | `.git/hooks/post-checkout` 비활성화 |
| WSL 워크트리 경로 에러 | `.git` 파일이 Windows 경로 참조 | 워크트리 아카이빙 후 재생성 |
| 팀메이트가 팀 소속 아님 | `team_name` 파라미터 누락 | `TeamCreate` 먼저 실행 확인 |
| tmux 세션 밖에서 실행 | tmux 세션 미생성 | `tmux new -s claude` 후 재시작 |
