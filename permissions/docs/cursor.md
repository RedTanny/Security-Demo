# Cursor permissions

This page is **Cursor only**. Claude Code: [claude.md](claude.md). Codex: [codex.md](codex.md).

Desktop and CLI use **different files**. Do not paste `Bash(…)` here — that is Claude Code.

A permission is an access grant. Name the command, not the family.

## Desktop — `.cursor/permissions.json`

| File | Applies to | Put it in git? |
| --- | --- | --- |
| `<repo>/.cursor/permissions.json` | Everyone who clones this repo | **Yes**, if the list is narrow. Same idea as Claude’s project `settings.json`. |
| `~/.cursor/permissions.json` | Only you, in every repo | **No.** It lives in your home directory. |

When both exist, Cursor **concatenates** the arrays. Team admin settings, if any, beat both.

| Field | What it is |
| --- | --- |
| `terminalAllowlist` | shell commands that run with no prompt. String match. Not a model. |
| `mcpAllowlist` | MCP tools that run with no prompt (`server:tool`). Same: not a model. |
| `autoRun.block_instructions` | English sentences for the **Auto-review classifier** only |

`block_instructions` are prompt-shaped. Cursor sends them to a second model that reviews a shell / MCP / Fetch call before it runs. They are **not** injected into the coding agent the way `.cursor/rules` is. They do nothing in Allowlist or Run Everything.

Treat them as a hint. The classifier can still allow a matching call; you can still approve it. Not a lock. The allowlist is the lock.

Matching is a **prefix**. `"git"` is every git command. `"git status"` is status (and anything that starts with `git status `).

| Wide | Specific |
| --- | --- |
| `"git"` | `"git status"`, `"git diff"`, `"git log"` |
| `"python"` | `"python src/app.py"` |
| `"curl"` | omit it |
| `"internal-docs:*"` or `"*:*"` | `"internal-docs:search"` |

Worked example: [`../cursor-specific.json`](../cursor-specific.json). Recommended baseline (desktop + sandbox + CLI): [`../cursor-manage.json`](../cursor-manage.json), [`../cursor-manage-sandbox.json`](../cursor-manage-sandbox.json), [`../cursor-cli-manage.json`](../cursor-cli-manage.json).

```json
{
  "mcpAllowlist": ["internal-docs:search"],
  "terminalAllowlist": [
    "python src/app.py",
    "pip install -r requirements.txt",
    "pytest tests/test_webhooks.py",
    "git status",
    "git diff",
    "git log"
  ],
  "autoRun": {
    "block_instructions": [
      "Any curl, wget, or nc command.",
      "Any command that reads .env, ~/.ssh, or ~/.aws."
    ]
  }
}
```

### Default when the file is missing

| Run mode | Unlisted terminal / MCP call |
| --- | --- |
| Auto-review | sandbox if it can; otherwise the classifier, then you |
| Allowlist | **ask** |
| Run Everything | runs — no prompt. This is the shortcut |

An empty `"terminalAllowlist": []` in the file is not “missing.” It means nothing is allowlisted.

## CLI — `.cursor/cli.json`

Cursor CLI is a **third syntax**. File: `~/.cursor/cli-config.json` (you) or `<repo>/.cursor/cli.json` (this repo).

| Token | Matches |
| --- | --- |
| `Shell(git)` | any `git` command |
| `Shell(curl:*)` | `curl` plus any args |
| `Read(.env*)` | env files |
| `Write(src/**)` | writes under `src` |
| `WebFetch(docs.github.com)` | that host |
| `Mcp(internal-docs:search)` | one MCP tool |

| List | Effect |
| --- | --- |
| `allow` | no prompt |
| `deny` | blocked — deny wins |

```json
{
  "permissions": {
    "allow": [
      "Shell(git)",
      "Read(src/**)",
      "WebFetch(docs.stripe.com)",
      "Mcp(internal-docs:search)"
    ],
    "deny": [
      "Shell(curl:*)",
      "Read(.env*)",
      "Write(**/.env*)"
    ]
  }
}
```

`Shell(git)` on the CLI is still a **family**. Prefer a tighter prefix when the CLI build supports `command:args` (for example `Shell(git:status*)` is still wider than one exact command). When in doubt, leave it off and **ask**.
