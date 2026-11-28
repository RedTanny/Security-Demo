# Agent tool permissions

A permission is an access grant. **Which file and which syntax** depend on the agent. Do not copy a rule from one handout into another.

| You use | Open this | Looks like |
| --- | --- | --- |
| Claude Code | [claude.md](claude.md) | `Bash(curl *)` in `.claude/settings.json` |
| Claude Code sandbox | [claude-sandbox.md](claude-sandbox.md) | `sandbox.enabled` — OS box around **Bash only** |
| Cursor | [cursor.md](cursor.md) | `"git status"` in `.cursor/permissions.json` |
| Codex | [codex.md](codex.md) | `sandbox_mode` in `config.toml` — not `Bash(…)` |

Recommended baseline to copy home (same *intent*, three syntaxes — do not paste across products):

| Agent | File | Goes to |
| --- | --- | --- |
| Claude Code | [`../claude-manage.json`](../claude-manage.json) | managed-settings.json, or `~/.claude/settings.json` |
| Cursor desktop | [`../cursor-manage.json`](../cursor-manage.json) + [`../cursor-manage-sandbox.json`](../cursor-manage-sandbox.json) | `.cursor/permissions.json` + `.cursor/sandbox.json` |
| Cursor CLI | [`../cursor-cli-manage.json`](../cursor-cli-manage.json) | `.cursor/cli.json` or `~/.cursor/cli-config.json` |
| Codex | [`../codex-manage.toml`](../codex-manage.toml) | `~/.codex/config.toml` |
| Codex org lock | [`../codex-manage-requirements.toml`](../codex-manage-requirements.toml) | `/etc/codex/requirements.toml` |

Same idea everywhere: name the command (or the box), not the family. Missing rules mean **ask**, not deny. “Don’t ask again” / Run Everything / `--yolo` is the shortcut this session is about.

“Don’t ask again” is not a review. Re-read the grant on a cadence.
