# claude-code-bootstrap

Claude Code skills & agents for bootstrapping dev environments from scratch.

Drop these into `~/.claude/skills/` (global) or `.claude/skills/` (project) and start using them.

## Skills

### `/team-setup` — Team Agent Orchestration

Set up PM(Lead) + teammates (BE/FE/QA) structure with a single command.

- Detects your environment (WSL / macOS / Windows)
- Installs missing dependencies or falls back gracefully
- Creates tmux layout or uses in-process mode
- Spawns teammates via `TeamCreate` + `Agent`
- Wires up `SendMessage` / `TaskList` communication

| Environment | Mode |
|-------------|------|
| WSL + tmux | tmux panes (recommended) |
| macOS + tmux | tmux panes |
| Windows native | in-process (Shift+Down) |
| No tmux | in-process fallback |

## Quick Start

```bash
# Copy to global skills
cp -r skills/team-setup ~/.claude/skills/

# Enable Agent Teams in ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Then in Claude Code:

```
/team-setup
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings
- tmux (optional, recommended)

## Installation

```bash
git clone https://github.com/jongmal/claude-code-bootstrap.git
cp -r claude-code-bootstrap/skills/team-setup ~/.claude/skills/
```

Or cherry-pick individual skill folders into your project's `.claude/skills/`.

## License

MIT
