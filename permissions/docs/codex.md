# Codex permissions

This page is **Codex CLI / IDE only**. Claude Code: [claude.md](claude.md). Cursor: [cursor.md](cursor.md).

Codex does **not** use `Bash(curl *)` or `"git status"` tokens. Do not paste those here.

Two layers:

| Layer | Question it answers |
| --- | --- |
| Sandbox | What can the command **do** (disk, network)? |
| Approval | When must Codex **ask** you? |

Default in a git repo: workspace write, **network off**, ask when leaving the box.

## Where the rules live

| File | Applies to | Put it in git? |
| --- | --- | --- |
| `~/.codex/config.toml` | Only you, in every repo | **No.** It lives in your home directory. |
| `<repo>/.codex/config.toml` | Everyone who clones this repo | **Yes**, if it has no secrets. |

Pick **one** shape in a given file. Do not mix `sandbox_mode` with `default_permissions`.

## Sandbox + approval (what most configs still have)

| Key | Typical values |
| --- | --- |
| `sandbox_mode` | `read-only` · `workspace-write` · `danger-full-access` |
| `approval_policy` | `on-request` · `never` |

| Intent | Config | What happens |
| --- | --- | --- |
| Default Auto | `workspace-write` + `on-request` | edits and commands in the repo; **ask** to leave the workspace or to use the network |
| Read-only | `read-only` + `on-request` | inspect; **ask** to write or escape the sandbox |
| The shortcut | `danger-full-access` + `never`, or `--yolo` | no sandbox, no prompts |

```toml
sandbox_mode = "workspace-write"
approval_policy = "on-request"

[sandbox_workspace_write]
network_access = false
```

`workspace-write` still keeps `.git`, `.codex`, and `.aws` **read-only** inside the repo.

### Network

Off by default. Turning it on without a proxy is **every host**.

| Setting | Effect |
| --- | --- |
| `network_access = false` | no outbound from commands |
| `network_access = true` and no proxy | network on, unrestricted |
| `network_access = true` **and** `features.network_proxy = true` plus a domain list | network on, only those hosts |

```toml
[features]
network_proxy = true

[sandbox_workspace_write]
network_access = true
```

Then put hosts on the proxy / profile allow list (`api.openai.com`, `docs.stripe.com`). A domain list with the proxy **off** does not restrict anything.

## Permission profiles (newer)

If the file has `default_permissions` instead of `sandbox_mode`, that is a **profile**.

| Built-in | Meaning |
| --- | --- |
| `:read-only` | commands cannot write |
| `:workspace` | write inside the workspace |
| `:danger-full-access` | the shortcut — same idea as `danger-full-access` |

Filesystem values: `read`, `write`, `deny`. Narrower `deny` wins over a broader write (so you can write the repo and still deny `**/*.env`).

```toml
default_permissions = "project-edit"

[features]
network_proxy = true

[permissions.project-edit]
extends = ":workspace"

[permissions.project-edit.filesystem.":workspace_roots"]
"**/*.env" = "deny"

[permissions.project-edit.network]
enabled = true

[permissions.project-edit.network.domains]
"docs.stripe.com" = "allow"
```

`network.enabled = true` without the proxy is unrestricted outbound. The domain table only counts when the proxy is on.

Worked example: [`../codex-specific.toml`](../codex-specific.toml)

## Default when nothing is set

| Folder | Codex starts as |
| --- | --- |
| Git repo | Auto: `workspace-write` + `on-request` |
| Not a git repo | `read-only` until you trust it |

Unlisted is not “curl is denied.” Curl is denied because **network is off**. Turn network on with no domain list and curl goes anywhere.

## Two traps

| Trap | What actually happens |
| --- | --- |
| Paste `Bash(curl *)` into `config.toml` | ignored — wrong product |
| `network_access = true` without `network_proxy` | the domain list is ignored; `curl` can reach any host |
