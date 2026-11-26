# MCP config — three grants

Same idea as [permissions](../../permissions/docs/README.md): the file is the grant. Connecting a server only **lists** the tools. Each call is still **ask** until a permission rule says otherwise.

Three cases, one fictional server (`internal-docs`). Fake tokens only. Do not start these servers.

| Case | Open this | Grant |
| --- | --- | --- |
| Personal, the bad way | [ex1](../ex1/README.md) | token pasted in JSON, `npx -y` floats to latest |
| Hygiene, not the standard | [ex2](../ex2/README.md) | env name + digest. Extra keys. Long-lived token still |
| Agent Plugins 1.0.0 | [ex3](../ex3/README.md) | closed schema, literal URL, credential stays in the client |

Schema constraints: [schema.md](schema.md).

## Where the file lives

| You use | Project (git) | Personal (not git) |
| --- | --- | --- |
| Claude Code | `.mcp.json` | `~/.claude.json` (`mcpServers`) |
| Cursor | `.cursor/mcp.json` | `~/.cursor/mcp.json` |
| Agent plugin | `mcp.json` at the plugin root | never — the client holds the secret |

`CODEOWNERS` on the project file. Personal files stay out of git. If a token was in a personal file and that file was committed, treat the token as leaked and rotate it.

Connecting the server is not the same decision as which tools a given session may call. Claude: `mcp__internal-docs__search`. Cursor: `mcpAllowlist` / `internal-docs:search`. See the permissions handouts.
