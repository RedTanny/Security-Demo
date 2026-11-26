# Claude Code permissions

This page is **Claude Code only**. Cursor: [cursor.md](cursor.md). Codex: [codex.md](codex.md).

A permission is an access grant. Name the command, not the family.
Missing rules mean **ask**, not deny and not allow.

## Where the rules live

| File | Applies to | Put it in git? |
| --- | --- | --- |
| `.claude/settings.json` | Everyone who clones this repo | **Yes.** Add and push it so the same grant travels with the code. |
| `.claude/settings.local.json` | Only your laptop. “Don’t ask again” writes here. | **No.** Add it to `.gitignore`. |
| `~/.claude/settings.json` | Only you, in every repo | Not in this repo. It lives in your home directory. |

“Commit it” just means `git add` + `git commit` + `git push` on `.claude/settings.json`, like any other project file. After that, a teammate who pulls the repo gets the same allow/deny list without copying anything by hand.

If an approved command contained a token, that token is now in `settings.local.json`. Treat it as leaked. Do not commit that file.

## Format

`Tool` or `Tool(specifier)`

| Rule | Matches |
| --- | --- |
| `Bash` | every shell command |
| `Bash(curl)` | only the exact command `curl` |
| `Bash(curl *)` | `curl` plus any arguments |
| `Bash(curl:*)` | same as `Bash(curl *)` |
| `Read(.env)` | that file |
| `WebFetch(domain:docs.stripe.com)` | that host only |
| `mcp__internal-docs__search` | one MCP tool (see below) |

`:*` is a **suffix**, not a field separator. It only means “and then anything,” and only at the end of the pattern. `Bash(git:* push)` does **not** match `git push`.

| You write | Means |
| --- | --- |
| `Bash(npm run build)` | that command only — not `--watch` |
| `Bash(npm run build *)` | `build` plus extra args |
| `Bash(npm run *)` | every npm script |
| `Bash(npm *)` | `npm install …` too — too wide |
| `Bash(git log *)` | `git log` only |
| `Bash(git *)` | every git command |

Put `*` after the subcommand.

## MCP tools (not `mcpAllowlist`)

There is no `mcpAllowlist` in Claude Code. That field is **Cursor**. Here, MCP tools are ordinary permission names: `mcp__<server>__<tool>`.

Connecting a server in `.mcp.json` only **lists** the tools. Each call is still **ask** until you put that name in `allow` or `deny`.

| Rule | Matches |
| --- | --- |
| `mcp__internal-docs__search` | one tool, `search`, on the `internal-docs` server |
| `mcp__internal-docs__*` | every tool on that server |
| `mcp__internal-docs` | same: every tool on that server |
| `mcp__*` | every MCP tool, every server — deny/ask only. An allow of `mcp__*` is skipped |

Do not put parentheses on these. `mcp__internal-docs__search(...)` is invalid; Claude Code ignores it. Dots in the server name become `_` (`my.server` → `mcp__my_server__*`).

Look the name up in `/permissions` — it must match what Claude Code shows, not Cursor’s `internal-docs:search`.

## Lists (deny wins)

| List | Effect |
| --- | --- |
| `deny` | blocked |
| `ask` | always prompt |
| `allow` | no prompt |

Order: **deny → ask → allow**. The first match wins. Specificity does not matter.

`deny: Bash(aws *)` plus `allow: Bash(aws s3 ls)` still **blocks** `aws s3 ls`. An allow cannot punch a hole in a deny.

## Default when `permissions` is missing

The session is Manual. Unlisted tools are **ask**.

| Tool | No rules |
| --- | --- |
| Read / Grep inside the project | allow |
| Read outside the project (`~/.ssh`, another repo) | ask |
| Read-only Bash (`ls`, `cat`, `pwd`, `git status`, `git diff`, …) | allow |
| Other Bash (`curl`, `python …`, `git push`) | **ask** |
| Edit / Write | **ask** |
| WebFetch, WebSearch, MCP | **ask** |

`Bash(curl:*)` not listed does not forbid curl. The first `curl` prompts. “Yes, and don’t ask again” then writes the rule into `.claude/settings.local.json`.

The mode that skips those prompts is `bypassPermissions` / `--dangerously-skip-permissions`.

## Wide vs specific

Same repo. Left column is the finding. Right column is the grant.

| Wide | Specific |
| --- | --- |
| `Bash(curl:*)` | put `Bash(curl *)` in **deny** |
| `Bash(git *)` | `Bash(git status)`, `Bash(git diff *)`, `Bash(git log *)` |
| `Bash(python *)` | `Bash(python src/app.py)` |
| `Bash(pytest *)` | `Bash(pytest tests/test_webhooks.py)` |
| `Read(**)` | omit it; **deny** `.env`, `~/.ssh/**`, `~/.aws/**` |
| `WebFetch(domain:*)` | `WebFetch(domain:docs.stripe.com)` |
| `mcp__internal-docs__*` | `mcp__internal-docs__search` |

Worked example: [`../claude-specific.json`](../claude-specific.json)

```json
{
  "permissions": {
    "allow": [
      "Bash(python src/app.py)",
      "Bash(pip install -r requirements.txt)",
      "Bash(pytest tests/test_webhooks.py)",
      "Bash(git status)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "WebFetch(domain:docs.stripe.com)",
      "mcp__internal-docs__search"
    ],
    "ask": ["Bash(git commit *)", "Bash(git push *)"],
    "deny": [
      "Bash(curl *)",
      "Bash(wget *)",
      "Bash(nc *)",
      "Read(.env)",
      "Read(.env.*)",
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)"
    ]
  }
}
```

## Two traps

| Trap | What actually happens |
| --- | --- |
| Deny vs allow | deny wins; write deny for exfil tools, allow for the few commands you still want when tired |
| Bash rule = the program | no — it matches the **text** Claude wrote. `Bash(curl *)` does not stop `/usr/bin/curl` or `sh -c 'curl …'`. Next layer: [claude-sandbox.md](claude-sandbox.md) |

The harness enforces these rules, not the model. A `CLAUDE.md` that says “run curl” still hits ask or deny.
