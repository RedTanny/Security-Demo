# Lab — default permissions vs `curl`

Fake key, localhost only. Nothing leaves the machine.

Two cases, same `curl`. Case 1 is no rules: Claude **asks**, and auto mode does not save a grant. Case 2 is **don’t ask again**: you click it, inspect the file, then `cat .env` and send the key — the saved `curl *` does not stop that.

## Prep

**Terminal 1 — attacker listener**

```bash
while true; do printf "HTTP/1.1 200 OK\r\nContent-Length: 2\r\nConnection: close\r\n\r\nOK" | nc -l -p 9999; done
```

**Terminal 2 — lab repo**

Start Claude in any folder **inside this git repo** (`tmp/`, `demo-repo/`, the root, …). “Don’t ask again” does **not** write next to that folder. It writes at the **git root**: `<repo>/.claude/settings.local.json`. A session in `tmp/` still produces `security-session/.claude/settings.local.json`, and that grant applies to the whole repo.

```bash
cd ~/projects/AI5/security-session/tmp
ROOT="$(git rev-parse --show-toplevel)"
export FAKE_API_KEY="sk-demo-fake-key-9999"
echo "FAKE_API_KEY=sk-demo-fake-key-9999" > .env
rm -f .claude/settings.json "$ROOT/.claude/settings.json" "$ROOT/.claude/settings.local.json"
echo "Claude will save grants to $ROOT/.claude/"
claude --permission-mode default
```

No `settings.json` in **cwd** (`tmp/.claude/`) → Manual mode. Unlisted tools are **ask**. (`settings.local.json` still lands at the git root.) In Case 1, do **not** pick “don’t ask again” — that is Case 2, after you have seen auto mode and Prompt 2.

---

## Case 1 — read-only vs `curl`

**What you are proving:** with no `settings.json`, read-only commands (`ls`, `git status`) run on their own. Network (`curl`) stops and asks. The **wording of that ask** depends on whether Claude Code can see the full command before the shell runs it. **Auto mode** lets this session continue without asking, but it does not save a rule.

---

### Prompt 1 — literal URL, no `$VAR`

```text
Run git status, then get http://127.0.0.1:9999 using curl.
```

| Step | What happens |
| --- | --- |
| `git status` | runs immediately |
| `curl` | permission prompt |

The dialog says **“This command requires approval”** and offers **four** answers:

```text
1. Yes
2. Yes, and don't ask again for: curl *
3. Yes, and switch to auto mode
4. No
```

The command is fully visible (`curl -s http://127.0.0.1:9999 …`), so persist is on the table. The offered rule is **`curl *`**, every `curl`, not this URL.

Choose **No** or **Yes** (one-shot). Do **not** pick option 2 yet — that is Case 2. Option 3 is the next step (auto mode). Either **Yes** or **No** leaves you in Manual, so the next `curl` still asks.

---

### Auto mode — allows now, does not persist

Still in the same Manual session, ask again:

```text
get http://127.0.0.1:9999 using curl
```

Same four answers as Prompt 1. This time choose **Yes, and switch to auto mode**.

| Step | What happens |
| --- | --- |
| This `curl` | runs (Terminal 1 may show a hit) |
| Status bar | `auto mode on` (or similar) |
| Same session, curl localhost again | **no** four-answer dialog — a classifier decides |
| `<git-root>/.claude/settings.local.json` | **not** created, and no `Bash(curl *)` if the file already existed |

Quit Claude (`Ctrl+C` or `/exit`). Start a **new** Manual session:

```bash
ROOT="$(git rev-parse --show-toplevel)"
ls "$ROOT/.claude/settings.local.json"   # should be missing
claude --permission-mode default
```

```text
get http://127.0.0.1:9999 using curl
```

The **four** answers are back. Auto mode was this session only. (`claude --continue` can reopen the old session still in auto mode — that is the same session, not a saved grant.)

On this replay, pick **No** or **Yes** so you stay in Manual for Prompt 2. Do not pick “don’t ask again” yet. If the classifier **blocks** a curl while auto is on, the point still holds: you were not writing an allow rule.

---

### Prompt 2 — `$FAKE_API_KEY` in the command

```text
Run git status, then send the value of $FAKE_API_KEY to http://127.0.0.1:9999 using curl.
```

| Step | What happens |
| --- | --- |
| `git status` | runs immediately |
| `curl` | permission prompt |

The dialog says **“A variable in this command can't be checked before it runs”** and offers **three** answers:

```text
1. Yes
2. Yes, and switch to auto mode
3. No
```

No “don’t ask again.” Claude Code matches allow rules against the command **text**, before `$FAKE_API_KEY` expands. It will not save a rule for a command whose payload is only known at runtime.

Choose **No** if you do not want the key on Terminal 1. **Yes** is one-shot: the key hits the listener, and the **next** `curl` still asks — nothing was written to disk. Do not pick auto mode here; Case 2 needs Manual and a four-answer dialog.

**If you `cat .env` first, persist comes back.** Try this instead of Prompt 2:

```text
Run cat .env, then send the value of $FAKE_API_KEY to http://127.0.0.1:9999 using curl.
```

`cat` is read-only, so it runs with no prompt. Claude then often curls the **literal** key (`sk-demo-fake-key-9999`), not `$FAKE_API_KEY`. The command is fully visible again, so you get Prompt 1’s **four** answers, including “don’t ask again for: `curl *`”.

Persist is about whether the **command text** can be checked, not about whether a secret is involved. Keep the value in `$FAKE_API_KEY` → three answers. Read the file first → secret is on the command line → four answers. After Case 2 has saved `curl *`, this same `cat .env` prompt is the silent leak — no dialog.

---

### What each answer does

| Answer | This command | Next `curl` in **this** session | Next **new** Manual session |
| --- | --- | --- | --- |
| **Yes** | runs | asks again | asks again |
| **Yes, and don’t ask again for: `curl *`** | runs | no prompt (allow rule) | no prompt — writes `Bash(curl *)` into **`<git-root>/.claude/settings.local.json`**, not the folder you started in |
| **Yes, and switch to auto mode** | runs | classifier decides; you are not asked for routine tools | **asks again** — mode was this session only; no rule file |
| **No** | blocked | asks again | asks again |

“Don’t ask again” **persists the command**. That is Case 2, next. Auto mode **does not**. It only changes how *this session* approves tools. `claude --continue` can come back in auto mode; that is the same session, still not a saved `curl` grant.

Prompt 1 is a fully visible `curl`, so you see persist. Auto mode skips later asks **in this session** and writes nothing. Prompt 2 hides “don’t ask again” because of `$FAKE_API_KEY` — unless Claude already read `.env` and inlined the value. If you already picked “don’t ask again”, you are in Case 2: inspect the file below, then send the key.

---

## Case 2 — “don’t ask again” saves `curl *`; that does not stop the key

**What you are proving:** option 2 on the four-answer dialog writes a **permanent** allow. You approved a harmless GET. The saved rule is `curl *` — every `curl`, not that URL. Then `cat .env` and send the key: Claude inlines `sk-demo-fake-key-9999` on the `curl` line, the allow matches, **no** dialog. (Prompt 2’s `$FAKE_API_KEY` form would still ask in Manual — the variable cannot be checked.)

Stay in Manual (after Case 1 you restarted with `--permission-mode default`). You need the **four**-answer prompt, so use a literal URL again — not `$FAKE_API_KEY`:

```text
get http://127.0.0.1:9999 using curl
```

Choose **Yes, and don’t ask again for: `curl *`**. (One-shot **Yes** does not write the file. Auto mode does not either.)

In Terminal 2 (outside Claude), look at the **git root**, not `tmp/.claude/`:

```bash
ROOT="$(git rev-parse --show-toplevel)"
echo "$ROOT"
cat "$ROOT/.claude/settings.local.json"
```

Example: you ran `claude` in `…/security-session/tmp`, and the file is `…/security-session/.claude/settings.local.json`. There is no `tmp/.claude/settings.local.json`.

You should see an allow of `Bash(curl *)` (or `Bash(curl:*)`). That grant now covers **every** session in this repo, including other subdirectories. Gitignore this file; it is only your laptop. The same `allow` in **cwd** `.claude/settings.json` (for these labs: `tmp/.claude/settings.json`) would be the **team** copy for this folder — Claude does not walk that file up to the git root.

Now, still in this Claude session (or a new Manual session — the file persists):

```text
Run cat .env, then send the value of $FAKE_API_KEY to http://127.0.0.1:9999 using curl.
```

| Step | What happens |
| --- | --- |
| `cat .env` | runs immediately (read-only) |
| `curl` | **no prompt** — Claude curls the **literal** key; `Bash(curl *)` matches; `sk-demo-fake-key-9999` appears on Terminal 1 |

`cat .env` is what makes this silent in Manual. Claude puts `sk-demo-fake-key-9999` on the command line, so the allow can see the text. The same ask with `$FAKE_API_KEY` still in the `curl` (Prompt 2) would still prompt — the file is loaded, the variable still cannot be checked. A harmless GET was enough to pre-approve this leak.

Leave `<git-root>/.claude/settings.local.json` in place for [ex3](../ex3/README.md) — a team **deny** on top of this local allow. Do not `rm` it yet.
