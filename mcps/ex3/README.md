# Lab — the standard way

Fake URL. Do not start this server. Handout: [schema.md](../docs/schema.md).

This is the Agent Plugins 1.0.0 document. Same fictional server as [ex1](../ex1/README.md) and [ex2](../ex2/README.md). Auth is not in the file.

```bash
cd ~/projects/AI5/security-session/mcps
diff -u ex2/mcp.json ex3/mcp.json
```

## Remote (preferred)

[mcp.json](mcp.json) — hosted Streamable HTTP. Real product twin: `https://security-mcp.api.redhat.com/mcp`.

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/mcp.schema.json",
  "mcpServers": {
    "internal-docs": {
      "type": "streamable-http",
      "url": "https://mcp.internal.example.com/docs"
    }
  }
}
```

| Field | Why |
| --- | --- |
| `$schema` | closed document. Unknown keys fail |
| `type: "streamable-http"` | not `"http"`, not inferred from `command` |
| literal `https://` URL | no `${HOST}`, no userinfo, no fragment |
| no `headers`, no `env`, no token | client does OAuth (`/mcp` login). Per-server, scoped, revocable |

**What you are proving:** there is nothing in this file to grep, commit, or leak. Revoke the grant in the client; you do not rotate a string in git.

Same validator as [ex2](../ex2/README.md) — ex1 and ex2 fail, these two pass:

```bash
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

**On stage:** open ex1 and ex3 side by side. Left file is two secrets. Right file is a URL.

**If someone asks** *Where do the OAuth tokens live if they aren't in `mcp.json`?* — In the client's auth store (like `~/.mcp-auth/` or the OS keychain) — which is why our `managed-settings.json` puts `~/.mcp-auth/**` on the `deny` list and the OS sandbox blocks reads to it.

## Local stdio (when the server cannot move)

[mcp.stdio.json](mcp.stdio.json) — same pin and env **name** as ex2, minus the extra keys, plus `$schema` and `type: "stdio"`.

The plugin names `INTERNAL_DOCS_API_TOKEN`. The client supplies the value. A pasted value here is still a token in a file the model can read.

`command` is one token (`podman`). Arguments stay in `args`.

## The skill names the tools

Connecting the server lists tools. It does not approve them. The closed list is [skill-grant.md](skill-grant.md): `search` and `list_collections`, not `mcp__internal-docs__*`. Pair with the permissions labs.

**On stage, then Benjamin:** pin the server. Read the allow rules on a cadence. “Don’t ask again” is not a review.
