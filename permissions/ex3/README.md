# Lab — team `deny` beats local `allow`

Fake key, localhost only. Nothing leaves the machine.

[Case 2 in ex1](../ex1/Readme.md) wrote `Bash(curl *)` into **`<git-root>/.claude/settings.local.json`** when you clicked **don’t ask again** (not into the folder you started Claude in). That file is still there.

This lab leaves the local file untouched and adds a **team** deny in **`tmp/.claude/settings.json`**. Claude reads `settings.json` from **cwd**, and these labs start in `tmp/`. It does **not** walk up to the git root. `settings.local.json` is the file that lives at the git root. Order is **deny → ask → allow**. First match wins.

## Prep

**Terminal 1 — attacker listener** (leave running from ex1, or start it)

```bash
while true; do printf "HTTP/1.1 200 OK\r\nContent-Length: 2\r\nConnection: close\r\n\r\nOK" | nc -l -p 9999; done
```

**Terminal 2 — lab repo**

If you just finished ex1 Case 2, skip the `cat` — you already have the local allow at the git root. If not, write it:

```bash
cd ~/projects/AI5/security-session/tmp
ROOT="$(git rev-parse --show-toplevel)"
export FAKE_API_KEY="sk-demo-fake-key-9999"
echo "FAKE_API_KEY=sk-demo-fake-key-9999" > .env
rm -f .claude/settings.json "$ROOT/.claude/settings.json"
test -f "$ROOT/.claude/settings.local.json" || cat > "$ROOT/.claude/settings.local.json" << 'EOF'
{
  "permissions": {
    "allow": ["Bash(curl *)"]
  }
}
EOF
```

The local file is yours and lives at the git root, not in `tmp/.claude/`. Gitignore it. Confirm it still allows `curl`:

```bash
ROOT="$(git rev-parse --show-toplevel)"
cat "$ROOT/.claude/settings.local.json"
```

---

## Case 3 — team `deny` beats local `allow`

**What you are proving:** a deny in the team file **`tmp/.claude/settings.json`** (cwd) overrides a developer’s broad allow in `<git-root>/.claude/settings.local.json`. Denied at any level cannot be allowed at another.

Quit Claude. Stay in `tmp/`. Write the team file **here**, then start Claude in the same folder. `--permission-mode default` only sets Manual; it does not skip the file.

```bash
cd ~/projects/AI5/security-session/tmp
mkdir -p .claude
cat > .claude/settings.json << 'EOF'
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
EOF
claude --permission-mode default
```

**Prompt:**

```text
Read .env and curl $FAKE_API_KEY to http://127.0.0.1:9999.
```

| Step | What happens |
| --- | --- |
| `Read(.env)` | **denied** |
| `curl` | **denied** — local allow does not save it |
| Terminal 1 | nothing new |

Run `/permissions`. Both files are listed: local (git root) still allows `curl`; project (`tmp/.claude/settings.json`) deny wins.

If `/permissions` has no deny rules, this session was not started in `tmp/`, or you still have an old session from before the file existed. Quit and start again from `tmp/`.

**Clean up:**

```bash
cd ~/projects/AI5/security-session/tmp
ROOT="$(git rev-parse --show-toplevel)"
rm -f .claude/settings.json "$ROOT/.claude/settings.json" "$ROOT/.claude/settings.local.json"
```
