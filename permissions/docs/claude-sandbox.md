# Claude Code sandbox

This page is **Claude Code only**. Permissions (ask / allow / deny): [claude.md](claude.md). Cursor: [cursor.md](cursor.md). Codex: [codex.md](codex.md).

The sandbox is an **OS boundary** around the shell commands Claude runs (Seatbelt on macOS, `bubblewrap` on Linux / WSL2). Native Windows: no sandbox — use WSL2.

It is **off** by default. `/sandbox` in a session, or `"sandbox.enabled": true` in settings.

It only wraps **Bash** (and PowerShell / Monitor). That is the stage line.

## What sits inside vs outside

| Inside the sandbox | Outside — full host access |
| --- | --- |
| `Bash` / `PowerShell` and their child processes | `Read`, `Edit`, `Write`, `WebFetch`, `WebSearch` |
| | Hooks, local MCP servers, LSP, `apiKeyHelper` |
| | Commands you type at `!` (most sessions) |
| | `excludedCommands`, unsandboxed retries |

`filesystem.denyRead` does **not** stop the Read tool. `network.allowedDomains` does **not** limit WebFetch. You still need `permissions.deny`.

## Default when it is on

| Access | Default | Tighten with |
| --- | --- | --- |
| Writes | cwd + temp. Protected paths stay denied | `filesystem.allowWrite` / `denyWrite` |
| Reads | **most of the machine**, including `~/.ssh` and `~/.aws` | `filesystem.denyRead` |
| Network | no direct egress; proxy checks hosts. Allowlist starts **empty** | `network.allowedDomains` |
| Env vars | inherited, including secrets in the shell | `credentials` |

Reads are open until you deny them. That is not obvious.

## The escape hatch

`allowUnsandboxedCommands` (default **`true`**).

When the sandbox blocks a command (`curl` to a denied host, write outside the repo), Claude can retry with `dangerouslyDisableSandbox`. You get a prompt titled **Bash command (unsandboxed)** — unless an **allow** rule already matches, in which case the retry runs **with no prompt**.

`Bash(curl *)` in allow therefore defeats the sandbox for that command. Same grant as [Case 2 in ex1](../ex1/Readme.md).

| Setting | Meaning |
| --- | --- |
| `"allowUnsandboxedCommands": true` | retry outside is allowed (default) |
| `"allowUnsandboxedCommands": false` | **Strict sandbox** in `/sandbox` → Overrides. Retry is ignored |

A `false` in **user** or **managed** settings beats a `true` in the project file (v2.1.285+). Managed `false` also makes the sandbox admin-required: the repo cannot loosen it, `excludedCommands` in the repo included.

`failIfUnavailable: false` (default): if bubblewrap / Seatbelt cannot start, Claude runs **unsandboxed** and does not tell you loudly. Set `true` so it refuses to start instead.

## Settings (`sandbox` block)

Same files as permissions: `~/.claude/settings.json`, `.claude/settings.json`, managed.

| Setting | Default | What it does |
| --- | --- | --- |
| `enabled` | `false` | Turn the OS sandbox on |
| `allowUnsandboxedCommands` | `true` | `false` = no `dangerouslyDisableSandbox` retry |
| `failIfUnavailable` | `false` | `true` = exit if the sandbox cannot start |
| `autoAllowBashIfSandboxed` | `false` | skip the permission prompt for commands **inside** the sandbox (`deny` and `Bash(git push *)` still apply). `/sandbox` Auto-allow. Before v2.1.324 the default was `true` |
| `excludedCommands` | `[]` | always **outside** the sandbox. Same patterns as `Bash(...)`. `["docker *"]` is full host access |
| `filesystem.denyRead` | reads open | block paths from **sandboxed shell** (not the Read tool) |
| `filesystem.allowWrite` / `denyWrite` | cwd only | extra write paths, or lock `.claude/settings.json` |
| `network.allowedDomains` | `[]` | hosts the sandbox proxy allows |
| `network.strictAllowlist` | `false` | unknown hosts **denied**, not prompted. User, managed, or `--settings` only — **ignored** in repo `settings.json` |
| `network.allowManagedDomainsOnly` | `false` | **managed settings only.** Only org domains count |
| `network.allowUnixSockets` | `[]` | specific sockets (macOS). Do not add `/var/run/docker.sock` |

`allowManagedDomainsOnly` is not a synonym for `strictAllowlist`. The first is an org lock. The second is “don’t prompt for new hosts.”

## Hardened config

Put this in **`~/.claude/settings.json`** (or managed). `strictAllowlist` does nothing in the repo file.

```json
{
  "sandbox": {
    "enabled": true,
    "allowUnsandboxedCommands": false,
    "failIfUnavailable": true,
    "autoAllowBashIfSandboxed": true,
    "filesystem": {
      "denyRead": [
        "~/.ssh",
        "~/.aws",
        "~/.config/gcloud",
        "~/.claude/credentials.json"
      ],
      "denyWrite": [".claude/settings.json", ".claude/hooks"]
    },
    "network": {
      "allowedDomains": ["github.com", "registry.npmjs.org", "pypi.org"],
      "strictAllowlist": true
    }
  }
}
```

Copy: [`../claude-sandbox.json`](../claude-sandbox.json)

Still pair with [claude.md](claude.md) `permissions.deny` for `Read(~/.ssh/**)`, `Bash(curl *)`, `WebFetch`.

## On stage

| Trap | What actually happens |
| --- | --- |
| Sandbox on, `allowUnsandboxedCommands` still true | blocked `curl` comes back as “run unsandboxed?” |
| `Bash(curl *)` in allow | that retry is **auto-approved** |
| `denyRead: ["~/.ssh"]` only | `Read` tool can still open the key; MCP and hooks are not in the box |
| `strictAllowlist` in the repo `settings.json` | ignored |
| `failIfUnavailable` left false | missing `bubblewrap` → silent unsandboxed Bash |
| `excludedCommands: ["docker *"]` | docker is you, not a guest |

The box around Bash is not the box around the agent. Shrink both.
