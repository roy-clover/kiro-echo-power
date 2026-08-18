# Validation guide — step by step

End-to-end check that (A) a hook fires in Kiro and (B) a power installs from
this GitHub repo. Takes about 10 minutes. Tested on Kiro IDE 1.0.406 / macOS.

Expected end state: **one** chat card named `Clover Security` per prompt, and a
proof line in `/tmp/clover-security-hook.log`.

---

## Part A — Prepare

**A1. Clear previous run artifacts**

```bash
rm -f /tmp/clover-security-hook.log /tmp/clover-security-hook-stdin.log
```

**A2. Confirm what is registered on disk**

```bash
ls ~/.kiro/hooks/                                   # should be empty (no global probes)
find ~/Code/kiro-echo-power/test-workspace -name '*.json' -o -name '*.md'
```

Expect exactly two files under `test-workspace`: `.kiro/hooks/echo-prompt.json`
and `.kiro/steering/clover-control.md`.

**A3. Confirm the hook's identity** — this is the only text a developer sees:

```bash
python3 -c "import json;d=json.load(open('$HOME/Code/kiro-echo-power/test-workspace/.kiro/hooks/echo-prompt.json'));[print(h['name'],'|',h['trigger']) for h in d['hooks']]"
```

Expect: `Clover Security | UserPromptSubmit`

---

## Part B — Validate the hook

**B1. Open the workspace** — Kiro → File → Open Folder →
`~/Code/kiro-echo-power/test-workspace`

**B2. Trust the workspace.** This is mandatory and is the single most common
cause of "nothing happens": in an untrusted workspace Kiro replaces every hook
trigger with a no-op and logs the suppression only at debug level — valid files,
zero fires, no visible error.

Command palette (`⌘⇧P`) → **Workspaces: Manage Workspace Trust** → **Trust**.

**B3. Reload the window** — `⌘⇧P` → **Developer: Reload Window**.

Required whenever hooks were *removed* (deleted hooks stay registered in a
running session) or when files under `.agents/plugins/` changed. Edits to
`.kiro/hooks/` alone are picked up live by the watcher.

**B4. Confirm the hook loaded.** Open the Output panel (`⌘⇧U`) and select the
Kiro agent log channel. Look for:

```
[KiroAgent] v2 hooks loaded 1 standalone hooks from .kiro/hooks/
```

- `1` → good, continue.
- `0` → the loader found no valid file. Check the same channel for
  `Hook file does not match v2 schema` or `Hook file is not valid JSON`.
- `hooks.v2.executionDisabledUntrustedWorkspace` → trust did not take; redo B2.

**B5. Send a prompt.** Any text, e.g. `hi`.

**B6. Observe the chat.** Expect **one** card while streaming:

```
▸ Run Command Hook   Clover Security
```

It collapses into a one-line summary (`… 1 hook triggered`) once the reply
arrives. The shell command and its output are never rendered — Kiro does not
include them for in-process hooks.

Seeing more than one card means stale probes are still registered → redo B3.

**B7. Confirm the hook actually ran**

```bash
cat /tmp/clover-security-hook.log
```

Expect: `[Clover Security] UserPromptSubmit fired at 2026-…Z`

**B8. Inspect the payload the hook received**

```bash
cat /tmp/clover-security-hook-stdin.log
```

Expect: `{"session_id":"sess_…","hook_event_name":"UserPromptSubmit","cwd":"…/test-workspace","prompt":""}`

Note `prompt` is **empty** on the IDE — a known IDE context gap. `session_id`
and `cwd` are reliable, and `cwd` is what gives git branch / remote / repo name.

---

## Part C — Validate the power install

**C1. Install from the GitHub URL.** Command palette →
**Powers: Configure** (`kiro.powers.configure`) → **Import power from GitHub** →
paste:

```
https://github.com/roy-clover/kiro-echo-power
```

A subdirectory URL also works: `…/tree/<branch>/<subdir>`.

**C2. Verify what the installer copied**

```bash
find ~/.kiro/powers/installed -type f
```

Expect **only** `POWER.md` and `steering/echo-power.md`. Everything else in the
repo — this guide, the README, `test-workspace/`, `dev.kiro/`, `plugin.json` —
is silently skipped. That is why the repo can carry its own documentation
without breaking installation.

**C3. Verify registration**

```bash
cat ~/.kiro/powers/installed.json
cat ~/.kiro/powers/registries/user-added.json
```

Expect an entry `{"name": "kiro-echo-power", "registryId": "user-added"}` and a
`source` block recording the clone URL and branch. A
`[PowersManager] mcp.json not found` warning is expected — this power ships no
MCP server.

**C4. Test whether the power's steering reaches the model.** Ask in chat:

```
what powers do you have available? activate clover-echo-power
```

| Reply contains | Conclusion |
|---|---|
| `[workspace-steering] loaded` **and** `[clover-echo-power] active` | Power steering reaches the model |
| only `[workspace-steering] loaded` | Workspace steering works; **power steering is inert** |
| neither marker | Steering is not reaching the model at all |

This is the open question: power activation is model-mediated (keyword-driven),
unlike hooks which fire deterministically at the engine level.

---

## Part D — Clean-room check from the repository

Proves a fresh machine gets the same result.

```bash
git clone https://github.com/roy-clover/kiro-echo-power /tmp/kiro-check
rm -f /tmp/clover-security-hook.log
```

Open `/tmp/kiro-check/test-workspace` in Kiro, trust it (B2), send a prompt, and
re-run B6–B8. Same single card, same log line.

---

## Part E — Clean up

```bash
rm -f /tmp/clover-security-hook.log /tmp/clover-security-hook-stdin.log
rm -rf /tmp/kiro-check
```

To uninstall the power: remove its entry from `~/.kiro/powers/installed.json`
and delete `~/.kiro/powers/installed/kiro-echo-power/`.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| No card, no log, no error | Workspace not trusted | B2 |
| `v2 hooks loaded 0 standalone hooks` | Schema or JSON error | Check the Kiro agent log channel for the parse warning |
| Card appears, log stays empty | Command failed after launch | Run the command by hand in a terminal; it must exit 0 |
| Chat stuck on "Working…" | Command blocks on stdin | Never `cat` stdin unbounded — use a bounded read (`read -t 1`) |
| More cards than hooks you expect | Deleted hooks still registered | Reload the window (B3) |
| `prompt` empty in the payload | Known IDE context gap | Use the Kiro CLI, or read the spec files / session store from disk |

## Notes for a production deployment

- The chat card **cannot be suppressed**. Kiro exposes six settings in total and
  the only display-related one is `kiroAgent.toolCardDisplayMode`
  (`collapseOnComplete` | `alwaysExpanded`) — neither hides hook cards, and the
  card event is emitted before the command runs. Control the hook's `name`, keep
  it to one hook, and it reads as a single branded line.
- Prefer a **standalone** `.kiro/hooks/*.json` hook over the
  `.agents/plugins/` form: the plugin loader generates its own name
  (`<dir>:<Event>-<i>-<j>`) which cannot be overridden.
- A power delivers **steering and MCP servers only**; hooks ship separately as a
  workspace `.kiro/hooks/` drop-in.
