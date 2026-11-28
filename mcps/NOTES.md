# MCP configs — projector crib

Share the handout in `docs/`. Three files, one server: do not mix the cases.

| Case | File | What you open |
| --- | --- | --- |
| Personal, the bad way | `ex1/mcp.json` | token in the JSON, `npx -y` |
| Hygiene, not the standard | `ex2/mcp.json` | env name + digest, extra keys |
| Agent Plugins 1.0.0 | `ex3/mcp.json` | `$schema`, `streamable-http`, no token field |

Do not start these servers. The files are the demo. Open ex1, grep it, then ex2, then ex3. Schema-check all four from `mcps/` (`schema.json`): ex1 and ex2 fail, ex3 and `mcp.stdio.json` pass.

Same sentence as the close: a token in the file is a finding. A scoped, revocable grant is a control.
