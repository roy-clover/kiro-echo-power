# clover-echo-power

A minimal Kiro **Power** that doubles as its own installation guide, plus a
hook-discovery experiment for the Clover-on-Kiro POC.

A Kiro power is just a Git repository: Kiro installs it by `git clone` and
copies in only `POWER.md`, `mcp.json`, and `steering/`. Everything else in the
repo — this README, the test harness, images — rides along as documentation
and is silently ignored by the installer. That means **this repo is both the
package and the guide**.

## Install (Kiro IDE)

1. Push this repo to any public GitHub location (or use it already pushed).
2. In Kiro: open the command palette → **`kiro.powers.configure`** (Powers:
   Configure), or focus the Powers panel.
3. Pick **"Import power from GitHub"** ("Provide a public GitHub URL to a
   custom power created by a community author").
4. Paste the repo URL:
   - Repo root: `https://github.com/<owner>/kiro-echo-power`
   - Or a subdirectory of any repo: `https://github.com/<owner>/<repo>/tree/<branch>/<subdir>`
     (sparse checkout — the power name is derived from the last path segment,
     lowercased/slugified to `[a-z0-9-]`).
5. Kiro clones the repo, copies `POWER.md` + `steering/` (+ `mcp.json` if
   present) into `~/.kiro/powers/installed/<name>/`, registers
   `{ "name": "<name>", "registryId": "user-added" }` in
   `~/.kiro/powers/installed.json`, and records the clone source
   (`repositoryCloneUrl`, `pathInRepo`, branch) in
   `~/.kiro/powers/registries/user-added.json`.

There is **no `kiro://` install deep link** (the scheme is OAuth-only) and no
marketplace publish flow — authors just host a repo; Kiro consumes it.
The recommended-powers registry Kiro shows by default is fetched from
`https://prod.download.desktop.kiro.dev/powers/default_registry.json`.

### Alternative: install from a local folder

The same QuickPick offers a folder import (`addCustomPowerByFolder`) — point it
at this directory. Or hand-install what the installer would keep:

```bash
mkdir -p ~/.kiro/powers/installed/clover-echo-power
cp POWER.md ~/.kiro/powers/installed/clover-echo-power/
cp -R steering ~/.kiro/powers/installed/clover-echo-power/
# deliberately ALSO copy the hook to test whether installed-dir hooks are read at all:
cp -R dev.kiro ~/.kiro/powers/installed/clover-echo-power/
```

Hand-installing also requires the `installed.json` entry above
(`{"name": "clover-echo-power", "registryId": "user-added"}` in
`installedPowers`).

## What a power may contain

Confirmed from the installed Kiro.app agent-extension bundle (package version
1.0.406, "registry-v2" powers implementation):

- The live GitHub install path (`installPowerFromRepository`) runs a
  **whitelist copy only** — `ALLOWED_FILES = ["POWER.md", "mcp.json"]`,
  `ALLOWED_DIRS = ["steering"]`. Extra files (README, LICENSE, `.gitignore`,
  images, test dirs, `dev.kiro/`) are **silently skipped, never rejected**.
- The strict validator that errors "Powers should only contain: POWER.md,
  mcp.json, and steering/*.md files" lives in a `PowerInstaller` class that
  **no reachable code path invokes** in this build — orphaned dead code.
- The real manifest is **`POWER.md` YAML frontmatter** (`name` required,
  `^[a-z0-9-]+$`; `description` required; optional `author`, `repository`,
  `license`, `tags`, `keywords`, `displayName`) — `plugin.json` appears
  nowhere in the bundle. Recommended body sections: `## Overview`,
  `## Available MCP Servers`, `## Tool Usage`, `## Configuration`.
- `KIRO_POWERS_HOME` relocates `~/.kiro/powers`.

## The experiment this repo carries

Question under test:

> Do hooks bundled inside a Kiro Power actually fire — and do they fire
> without keyword activation?

**Static answer (same bundle): no.** Power installs never copy a hook file,
and the hook engine never reads the powers directory. Hooks load only from:

- `<workspace>/.kiro/hooks/*.json` — watched, unified v1 schema; and
- `<workspace>/.agents/plugins/<plugin>/hooks/hooks.json` — an "Open Plugin"
  loader (telemetry tag `open-plugin`) using the **Claude-Code plugin hooks
  schema** (`{"hooks": {"UserPromptSubmit": [{"matcher": ..., "hooks":
  [{"type": "command", ...}]}]}}`, event aliases
  `UserPromptSubmit`/`userPromptSubmit`/`promptSubmit`, default timeout 60s).

No home-level (`~/.kiro/hooks`) path exists in this build. Hook exit-code
semantics: exit 0 → stdout forwarded to context for
SessionStart/UserPromptSubmit/PreToolUse; exit 2 → block (stderr forwarded);
other → silent failure.

So the expected empirical result is **never-fires** for the power-bundled hook
(dropped at install time), while the workspace hook and the open-plugin hook
fire on every prompt. The scaffold still tests all three so the conclusion is
empirical, not just static.

**Clover POC implication:** a Power delivers steering + MCP servers only; the
enforcement hooks ship per workspace via `.kiro/hooks/` or the
`.agents/plugins/` Claude-schema route (nearly identical to the existing
clover-claude-plugin layout).

## Layout

```
kiro-echo-power/
├── plugin.json                  # blog-era manifest GUESS — NOT read by the IDE (kept to test whether anything consumes it)
├── POWER.md                     # the real (required) manifest: YAML frontmatter + steering body
├── steering/echo-power.md       # allowed power payload; proves power install/activation
├── dev.kiro/hooks/echo-prompt.json   # the power-bundled hook under test (v1 schema)
├── test-workspace/              # open THIS folder in Kiro for the control runs
│   ├── .kiro/hooks/echo-prompt.json                        # control 1: plain workspace hook
│   └── .agents/plugins/clover-echo-plugin/hooks/hooks.json # control 2: open-plugin hook (Claude-style schema)
└── README.md
```

Log files (all under `/tmp`, one `.log` proof line per fire plus a `-stdin.log`
capturing the JSON payload):

| Condition | Proof log | Stdin capture |
|---|---|---|
| Power-bundled hook | `/tmp/kiro-echo-power.log` | `/tmp/kiro-echo-power-stdin.log` |
| Workspace hook | `/tmp/kiro-workspace-hook.log` | `/tmp/kiro-workspace-hook-stdin.log` |
| Open-plugin hook | `/tmp/kiro-open-plugin-hook.log` | `/tmp/kiro-open-plugin-hook-stdin.log` |

## If hooks don't fire (zero logs) — read this first

Confirmed from the agent runtime bundled in Kiro 1.0.406 (`@kiro/agent`, the
"v2" hook engine — hardcoded on, no setting):

- The v2 engine reads **`<workspace>/.kiro/hooks/*.json`** and
  **`~/.kiro/hooks/*.json`** in the `{"version":"v1","hooks":[...]}` format
  used here, plus `.agents/plugins/<dir>/hooks/hooks.json`. (The `*.kiro.hook`
  when/then files are a separate legacy system — ignore it.)
- **Workspace trust is the silent kill switch.** In an untrusted workspace,
  every hook trigger is replaced with a no-op and the suppression is logged
  only at debug level — valid files, zero fires, zero visible errors.
  Fix: Command palette → "Workspaces: Manage Workspace Trust" → Trust.
- `.agents/plugins/` is scanned only when a session is built — **start a new
  chat session** after adding files (editing anything under `.kiro/hooks/`
  also triggers a reload of everything).
- No approval click is needed for v2 command hooks; commands spawn with
  `shell: true`, cwd = first workspace root, hook context JSON on stdin.
- Load diagnostic: Output panel → Kiro Agent channel → look for
  `[KiroAgent] v2 hooks loaded N standalone hooks from .kiro/hooks/` and
  `... N Open Plugin hooks`. `N = 0` → wrong root or schema warning
  (`Hook file does not match v2 schema` / `not valid JSON` in the same
  channel); `N > 0` but no execution → workspace trust.

## Validation checklist

1. Clear old logs: `rm -f /tmp/kiro-echo-power*.log /tmp/kiro-workspace-hook*.log /tmp/kiro-open-plugin-hook*.log`
2. Install the power (above). Open `test-workspace/` in Kiro.
3. Send any prompt **without** the activation keywords (no "echo"/"clover-test"), e.g. "hi".
   - Check all three `/tmp/*.log` files.
4. Send a prompt **with** a keyword, e.g. "echo test — are you active?".
   - Expect `[clover-echo-power] active` / `steering loaded` in the reply if the power activated.
   - Re-check `/tmp/kiro-echo-power.log`.
5. Conclusion matrix for the power-bundled hook:

| Observation | Conclusion |
|---|---|
| `/tmp/kiro-echo-power.log` grows on every prompt | Power hooks fire always (persistent registration) |
| grows only after a keyword prompt | Power hooks fire only while keyword-activated |
| never grows, but workspace/open-plugin logs grow | Powers don't carry hooks (static analysis says THIS) |
| no log grows at all | Hooks aren't firing at all — fix the control first |

## Remaining uncertainties

- `plugin.json` and `dev.kiro/hooks/` come from the launch-blog description of
  Powers; neither exists anywhere in the IDE bundle. They may belong to the
  Kiro CLI (not installed on this machine) or to a newer build.
- The open-plugin hooks.json shape was reconstructed from the minified loader;
  `matcher` is optional per entry and UserPromptSubmit ignores tool matchers.
