# 🚀 claude-code-bootstrap

> Claude Code로 개발 환경을 처음부터 뚝딱 세팅하는 스킬 & 에이전트 모음

`~/.claude/skills/` (글로벌) 또는 `.claude/skills/` (프로젝트)에 복사하면 바로 사용 가능합니다.

## 📦 Skills

### [`/team-setup`](skills/team-setup/) — 팀 에이전트 오케스트레이션

PM(Lead) + 팀메이트(BE/FE/QA) 구조를 한 방에 셋업합니다.

- 🔍 환경 자동 감지 (WSL / macOS / Windows)
- 📦 누락 의존성 자동 해결 or 폴백
- 🖥️ tmux 레이아웃 자동 구성
- 🤖 `TeamCreate` + `Agent` spawn
- 💬 `SendMessage` / `TaskList` 통신 연결

| 환경 | 모드 |
|------|------|
| 🐧 WSL + tmux | tmux pane (권장) |
| 🍎 macOS + tmux | tmux pane |
| 🪟 Windows native | in-process (Shift+Down) |
| 🚫 tmux 없음 | in-process 폴백 |

## ⚡ Quick Start

```bash
# 스킬 복사
git clone https://github.com/jongmal/claude-code-bootstrap.git
cp -r claude-code-bootstrap/skills/team-setup ~/.claude/skills/
```

Agent Teams 활성화 (`~/.claude/settings.json`):
```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Claude Code에서:
```
/team-setup
```

끝! 🎉

## 📋 Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings
- tmux (optional, recommended)

## 📄 License

MIT
