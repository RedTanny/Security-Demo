# Lab — personal MCP, the bad way

Fake tokens only. Do not start these servers. Nothing in this file is a live credential.

This is the file people paste from a quickstart into `~/.claude.json` or `~/.cursor/mcp.json`. Same shape as `demo-repo/.mcp.json`. GitGuardian’s 2025 pass found **24,008** unique secrets in MCP-related configs on public GitHub; official READMEs taught this layout.

## Prep

Open the file. Then grep it, the way Beat 1 of the demo repo does:

```bash
cd ~/projects/AI5/security-session/mcps
grep -nE 'TOKEN|npx|-y' ex1/mcp.json
```

| Line | Finding |
| --- | --- |
| `GITHUB_PERSONAL_ACCESS_TOKEN": "demo-gh-…"` | personal PAT in a file the agent reads |
| `INTERNAL_DOCS_API_TOKEN": "demo-internal-…"` | same, second server |
| `"npx"` + `"-y"` | unpinned. Tomorrow’s package is today’s approval |
| `@modelcontextprotocol/server-github` | archived/deprecated (replaced by `github/github-mcp-server`). `npx -y` still fetches it |
| no `type`, no `$schema` | client infers stdio. Schema would reject the whole document |

On a real laptop this file lives in the **home** directory, so every repo inherits both servers. If someone copies it into the project and pushes, the tokens are public. Rotate them.

---

## Case 1 — the file is the secret

**What you are proving:** a personal MCP config with a pasted token is a finding even before any tool runs. `npx -y` is a supply chain you re-approve every morning without looking.

**On stage:** do not run `npx`. Point at the two env values, then at `-y`. Say: this is what the scanner flags. Connecting the server also lists **every** tool it exposes — there is no tool list in this file.

The GitHub entry is the personal-use half: a PAT with the developer’s orgs, not a bot grant. The docs entry is the same bug on an internal API. Both are readable by the agent on disk, and active in the server process the moment it connects.

**Do not** copy this file into `~/.claude.json` or `~/.cursor/mcp.json`.

Next: [ex2](../ex2/README.md) — token out, still not the standard.
