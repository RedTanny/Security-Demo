# Agent tool permissions

A permission is an access grant. **Which file and which syntax** depend on the agent. Do not copy a rule from one handout into another.

| You use | Open this | Looks like |
| --- | --- | --- |
| Claude Code | [claude.md](claude.md) | `Bash(curl *)` in `.claude/settings.json` |
| Claude Code sandbox | [claude-sandbox.md](claude-sandbox.md) | `sandbox.enabled` — OS box around **Bash only** |
| Cursor | [cursor.md](cursor.md) | `"git status"` in `.cursor/permissions.json` |
| Codex | [codex.md](codex.md) | `sandbox_mode` in `config.toml` — not `Bash(…)` |

Same idea everywhere: name the command (or the box), not the family. Missing rules mean **ask**, not deny. “Don’t ask again” / Run Everything / `--yolo` is the shortcut this session is about.

“Don’t ask again” is not a review. Re-read the grant on a cadence.
