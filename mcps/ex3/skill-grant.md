# The other half of the grant — named tools, one ecosystem

`mcp.json` connects the server. It does not say which tools this workflow may call. That list lives on the skill. Shape copied from `cluster-creator` (Assisted Installer). Fake server name here is `internal-docs`.

```yaml
name: docs-search
allowed-tools: search list_collections
metadata:
  mcp_server: internal-docs
  environment_vars:
    - INTERNAL_DOCS_API_TOKEN
```

| Wide | The grant you ship |
| --- | --- |
| one MCP entry that can see every API you have a token for | one server for one ecosystem |
| `tools: *` / no tool list / `mcp__internal-docs__*` | the skill names `search` and `list_collections` |
| token pasted into `mcp.json` | variable named, value in the environment or OAuth in the client |
| `npx -y some-server` | pinned digest, or a literal `https://` URL |

`cluster-creator` is the real example: it uses `openshift-self-managed` (Assisted Installer API) and does **not** take `openshift-ocm-managed` (ROSA / ARO / OSD). Different API, different tools. An agent installing a self-managed cluster is not handed the managed-cloud tool set “because both are OpenShift.”

Never print credential values.
