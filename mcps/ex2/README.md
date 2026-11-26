# Lab — hygiene, not the standard

Fake values only. Do not start this server. The digest is a placeholder.

This is the afternoon fix on a laptop: secret out of the file, package pinned. It is what `ocp-admin/mcps.json` looked like before the Agent Plugins schema — and what a careful `.mcp.json` in a repo looks like today.

Diff against [ex1](../ex1/mcp.json):

```bash
cd ~/projects/AI5/security-session/mcps
diff -u ex1/mcp.json ex2/mcp.json
```

## Prep

```bash
# value in the shell, not in the JSON
export INTERNAL_DOCS_API_TOKEN="demo-internal-docs-token-0000"
grep -nE 'TOKEN|sha256|description|security' ex2/mcp.json
```

| What changed | Still true |
| --- | --- |
| no literal token | `${INTERNAL_DOCS_API_TOKEN}` is a name. The value is in the environment |
| `podman` + digest, not `npx -y` | pin is real; the image is still a local process with a long-lived token |
| GitHub PAT is gone | one server, not a personal dump |
| `description` / `security` | homemade keys. Clients ignore them. The schema **rejects** the entry |

---

## Case 2 — clean tree, closed standard

**What you are proving:** moving the secret to the environment and pinning the image is necessary and not sufficient. This file would still fail [schema.md](../docs/schema.md): missing `$schema`, no `type`, extra keys.

**On stage:** grep is clean of literal tokens. Then point at `description` and `security`. Say: an extra key is not documentation, it is invalid under 1.0.0. The credential is still a long-lived env var, not a revocable grant.

### Trap — “we switched to HTTP”

This is not in `mcp.json`. It is the upgrade that still fails the standard:

```json
{
  "internal-docs": {
    "type": "http",
    "url": "https://${DOCS_MCP_HOST}/mcp",
    "headers": {
      "Authorization": "Bearer ${INTERNAL_DOCS_API_TOKEN}"
    }
  }
}
```

| Looks like | Schema says |
| --- | --- |
| `type: "http"` | must be `streamable-http` (or `sse`) |
| `${DOCS_MCP_HOST}` in the URL | URL is a literal. Variables do not expand there |
| `Authorization: Bearer …` | headers are package data. No secrets. Variables do not expand there |

That shape is how some product JSON still ships today. The standard rewrite is either a stdio proxy that reads the token from the environment, or a remote URL with **no** header secret and OAuth in the client. That is [ex3](../ex3/README.md).
