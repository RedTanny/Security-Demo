# Agent Plugins MCP runtime (1.0.0)

This page is the **standard** `mcp.json`. Claude/Cursor will happily load [ex1](../ex1/mcp.json) and [ex2](../ex2/mcp.json). The schema will not.

Canonical: [mcp.schema.json](https://agent-plugins.org/schemas/1.0.0/mcp.schema.json). Authors: [MCP servers](https://agent-plugins.org/plugin-authors/mcp-servers). Clients: [MCP runtime](https://agent-plugins.org/client-implementers/mcp-runtime).

## Closed document

Top level is `$schema` and `mcpServers` only. Each server has a `type`. `additionalProperties` is false. An unknown key — including a homemade `token`, `apiKey`, `description`, or `security` — makes that entry **invalid**.

| Transport | `type` | Required | Optional |
| --- | --- | --- | --- |
| Local process | `stdio` | `command` | `args`, `env`, `cwd` |
| Remote (current) | `streamable-http` | `url` | literal `headers` |
| Remote (deprecated) | `sse` | `url` | literal `headers` |

`type: "http"` is Claude/Cursor shorthand. It is not in the schema.

## What may not be in the file

- **No credential field.** 1.0.0 defines no OAuth field and no secret-reference field. Authentication is the client’s job.
- **Headers are package data.** They must not contain a secret. Plugin variables do **not** expand in headers.
- **The URL is a literal.** Absolute `http` or `https`, no userinfo, no fragment. Non-loopback is `https`. Variables do **not** expand in the URL. The host that was reviewed is the host the client opens.
- **`command` is one executable token**, not a shell string. Arguments travel in `args`.

## Local vs remote, spoken

| | Local stdio | Remote `streamable-http` |
| --- | --- | --- |
| Secret | environment, named in `env`, value not in the package | client OAuth. Nothing in the JSON |
| Pin | image digest or binary version | the URL is the pin |
| What the plugin ships | command + args + env **names** | the URL |

A value you paste into `env` is still a token in a file the model can read. The plugin names the variable. The client supplies the value.

Worked files: [ex3/mcp.json](../ex3/mcp.json) (remote) and [ex3/mcp.stdio.json](../ex3/mcp.stdio.json) (local).

## Live check

[schema.json](../schema.json) is that closed document plus the URL/variable rules the official JSON Schema leaves as prose (`https://`, no `${...}`, no userinfo, no fragment). `headers` stay optional package data; values must not contain `${`. `check-jsonschema` is not on this box — `python3` + `jsonschema` is.

```bash
cd ~/projects/AI5/security-session/mcps
python3 - <<'PY'
import json
from jsonschema import Draft202012Validator
from jsonschema.exceptions import best_match
schema = json.load(open("schema.json"))
validator = Draft202012Validator(schema)
for path in ["ex1/mcp.json", "ex2/mcp.json", "ex3/mcp.json", "ex3/mcp.stdio.json"]:
    error = best_match(validator.iter_errors(json.load(open(path))))
    print(f"{path}: {'FAIL — ' + error.message if error else 'PASS'}")
PY
```

ex1 and ex2 fail. ex3 and `mcp.stdio.json` pass.
