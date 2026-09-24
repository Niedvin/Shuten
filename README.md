# Shuten

Two plugins that cut an AI agent's excess text: **Shut** — what it writes to you in the chat,
**CCShut** — what it leaves as comments in your code. Both load at session start, so they
apply from the first message rather than from the moment the agent decides they are relevant.

English version: the question, the warning and the final answer are plain English.
Ukrainian version — [Shutuk](https://github.com/Niedvin/Shutuk).

They install into Claude Code and Claude Desktop as plugins, and into Codex, opencode,
Gemini CLI and Cursor as skills.

---

## What they actually do

### Shut — token savings

The problem: the agent spends your tokens on text that changes nothing — reports on its own
process, announcements of "now I will read the file", restatements of your request, half-screen
summary tables. You pay for it and do nothing with it.

Shut allows exactly four shapes of message and nothing else:

| Shape | Register | Rule |
|---|---|---|
| Step label | telegraphic | ≤ 4 words, no comma or dash inside, most often absent entirely |
| Question | plain | only when your request can be read two ways and the two would give different results |
| Warning | plain | only when you must decide or act right now |
| Answer | plain | ≤ 8 lines, no tables, no headings, no list of the steps taken |

No running commentary, no announcing a tool call right before making it, no "run it and see
how it works". Numbers as digits, relations as symbols (`>`, `≈`, `→`, `∵`).

**It does not touch the code or the result of the work.** The rule covers only what the agent
writes to you.

Hooks re-assert the rule after every skill load and check the turn before it closes, so it
does not dissolve over a long session.

### CCShut — against context rot

The problem: the agent buries a file in comments that explain what the code already says. A
few edits later the code has moved on and the comment has not — now it lies. Every later read
of that file, by you or by the agent, costs tokens and points the wrong way. That is context
rot.

A comment is written only if it passes both filters:

1. **Permanence** — it stays true after the code around it is rewritten.
2. **Irreducibility** — a senior engineer already working in this repo could not recover it
   from the names, the types, the control flow, or one call site away.

Plus: one line per comment, ending in the date it was written (` — 2026-09-05`), under 5% of a
file's lines, with no exception for doc comments. Comments already sitting in a file the agent
edits go through the same filters — the ones that fail are deleted. TODO, FIXME and notes meant
for you go into the reply, not into the file.

A `PostToolUse` hook checks every file written and hands the agent back a list of violations.

---

## Installation

### Claude Code and Claude Desktop — through the marketplace

```
/plugin marketplace add Niedvin/Shuten
/plugin install ccshut@shuten
/plugin install shut@shuten
```

Your own GitHub repo works as a marketplace with no registration: Claude clones it itself and
reads `.claude-plugin/marketplace.json`.

**Auto-update.** Third-party marketplaces start with it off. Turn it on once: `/plugin` →
**Marketplaces** → `shuten` → **Enable auto-update**. Or in `~/.claude/settings.json`:

```json
"extraKnownMarketplaces": {
  "shuten": { "source": { "source": "github", "repo": "Niedvin/Shuten" }, "autoUpdate": true }
}
```

Claude Code then checks GitHub after each launch and pulls every new commit; `/reload-plugins`
loads it without a restart. The plugins carry no `version` field on purpose — the commit is
the version, so every push reaches users. By hand: `/plugin marketplace update shuten`.

### Claude Desktop — by drag and drop

Download [`dist/CCShut.zip`](dist/CCShut.zip) and [`dist/Shut.zip`](dist/Shut.zip) and drop
each into the plugin upload window. One zip per plugin; `.claude-plugin/plugin.json` sits at
the root of each, which is exactly what the uploader looks for.

### Codex, opencode, Gemini CLI, Cursor — with the installer

The marketplace exists only in Claude. For the rest, clone the repo and run the pair for your
OS. Both pairs do the same thing and leave byte-identical files.

```
git clone https://github.com/Niedvin/Shuten
cd Shuten
```

| OS | Install | Uninstall | Needs |
|---|---|---|---|
| Windows | double-click `install.cmd`, or `powershell -ExecutionPolicy Bypass -File install.ps1` | `uninstall.cmd` | nothing — PowerShell 5.1 ships with Windows 10/11 |
| macOS | `bash install.sh` | `bash uninstall.sh` | nothing — bash, unzip and osascript ship with macOS |
| Linux | `bash install.sh` | `bash uninstall.sh` | `node`, for the JSON edits |

The installer finds which agents are on the machine and skips the rest. `--dry-run` prints the
whole plan and changes nothing — run that first if you want to see the list.

---

## What the installer touches, and why

It edits files in your home directory. Nothing runs as administrator, nothing goes out over
the network, and every file it changes is copied next to itself first.

| Path | What happens | Why |
|---|---|---|
| `~/.claude/skills/{ccshut,shut}/` | the plugin is copied in | Claude Code looks for plugins here and nowhere else; without the copy the rule simply never switches on |
| `~/.codex/skills/`, `~/.config/opencode/skills/`, `~/.gemini/skills/`, `~/.cursor/skills-cursor/`, `~/.agents/skills/` | a flat `SKILL.md` is copied in | every agent has its own skills path and there is no shared one — hence a copy in each |
| `~/.codex/hooks.json` + `~/.codex/hooks/shut-*` | two `SessionStart` entries are added | a skill loads only once the agent decides it is relevant, and a rule about how to talk is needed from the first line. The hook hands it over at the start of every session. Other hooks in the file stay where they are |
| `~/.config/opencode/AGENTS.md`, `~/.gemini/GEMINI.md` | a marked block is added | these two agents have no session-start hook. The only way to give them the rule every time is to put the text in a file they read on their own |
| `~/.claude/settings.json` | `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION=false`, `CLAUDE_CODE_ENABLE_AWAY_SUMMARY=0` | these are next-prompt suggestions and an auto-summary after a pause. Both print text of their own, you pay for it, and Shut bans it |
| `~/.claude/settings.local.json` | `outputStyle: Shut` | a built-in output style can drop the brevity rules; this one repeats them, so they do not go missing |
| `~/.claude/output-styles/shut.md` | installed | the style file itself; without it the line above points at nothing |
| every project `.claude/` directory found | the same two settings | Claude reads project settings on top of the global ones, so a project can quietly turn back on what you just switched off |
| the `language` key in those settings | **removed** | any value there adds "Always respond in \<lang\>" to every explanation. Even the 4-word step label switches language and the four shapes stop holding — whatever language it names |
| `~/.shut-backups/` | originals and state | copies of every file before it changed, plus a list of what exactly changed; `uninstall` reads it and puts everything back |

**Why it needs disk access.** Finding project `.claude` directories is the only step that
leaves the home directory: it walks `$HOME` plus every local drive (Windows) or every volume
under `/Volumes` (macOS) four levels deep, skipping `Library`, `AppData`, `node_modules`, build
directories and everything starting with a dot. It reads directory names, opens only files
named `settings.json` / `settings.local.json`, and writes only to those. Removable and network
drives are not touched at all. `--dry-run` shows the exact list before anything is written, and
`--only claude` keeps everything to the global config.

On macOS, `~/Desktop`, `~/Documents` and `~/Downloads` are hidden from the terminal without
Full Disk Access. Projects under them are simply skipped; the installer names those directories
and says so. Granting access is optional.

---

## Uninstalling

```
uninstall.cmd                   # Windows
bash uninstall.sh               # macOS / Linux
/plugin uninstall ccshut@shuten # Claude Code, if you installed through the marketplace
```

Run the script from the same directory you cloned the repo into — `uninstall.cmd` and
`uninstall.sh` live there. If you have already deleted it, cloning again is enough: the
installer records what it changed in `~/.shut-backups/uninstall.json`, not in the repo
directory, so a fresh clone sees the same state.

It removes only what the installer put there: skill directories carrying the
`.shut-install.json` marker (`--force` for the rest), its own entries in `~/.codex/hooks.json`,
the marked blocks, and every changed setting — restored from `~/.shut-backups/uninstall.json`.
Every file it edits is copied to `*.bak-uninstall` first.

---

## Options

The same set in both installers; PowerShell spells them as switches.

```
--dry-run       -DryRun         print the plan, change nothing
--list          -List           show the agents found
--only claude   -Only claude    one agent (comma-separated for several)
--skip gemini   -Skip gemini    skip one
--no-always-on  -NoAlwaysOn     skills and hooks only, leave AGENTS.md alone
--no-hooks      -NoHooks        skills and AGENTS.md only, no Codex hook
--keep-caveman  -KeepCaveman    leave the caveman skill enabled
```

Uninstall takes `--force` / `-Force` and `--keep-caveman-off` / `-KeepCavemanOff`.

---

## Notes

**Trust in Codex.** Codex will not run a new hook until you trust it. Run `/hooks` inside
Codex, trust the two `shut-` entries, then run the installer again — it sees the trust and
removes the now-duplicated block from `AGENTS.md`. Until then both are in place, so the rule
never lapses.

**Caveman.** If the `caveman` skill is installed, it gets disabled: two rewriters of the same
answer is not a defined state. What was done is recorded in
`~/.shut-backups/caveman-state.json` and restored by `uninstall`. `--keep-caveman` skips the
step. Lines you wrote yourself like "use /caveman" in your own `CLAUDE.md` are named by the
installer but not edited.

**macOS.** The hook JSON ships prebuilt inside the plugins, so the rule works on a Mac with no
Python at all — it is needed only by the three checks that score a turn, and without it they
exit quietly. The hook scripts take `python3` and skip `/usr/bin/python3` until `xcode-select
-p` succeeds: that path is a stub that pops open the Xcode CLT installer.

**Project files.** Both plugins carry the whole rule, so a project `CLAUDE.md` / `AGENTS.md`
does not need to restate it. A project may add to these rules — it may never loosen them.

**Re-running** updates in place and leaves alone anything already correct. Whatever was sitting
at a target path moves into `~/.shut-backups/` rather than being deleted.

**Changing the language** is a matter of editing `plugins/shut/hooks/context.md` and
`plugins/shut/always-on.md` — or just taking [Shutuk](https://github.com/Niedvin/Shutuk), where
those three shapes are Ukrainian.

---

## Repo layout

```
.claude-plugin/marketplace.json   the marketplace manifest
plugins/ccshut/                   the CCShut plugin (.claude-plugin/, hooks/, skills/)
plugins/shut/                     the Shut plugin
dist/*.zip                        those same two plugins, packed for drag and drop
install.ps1 / uninstall.ps1       the Windows installer, plus .cmd wrappers
install.sh / uninstall.sh         the macOS and Linux installer
shut-json.js                      the JSON editor the shell installer runs through osascript or node
shut.md                           the Claude Code output style
```

`always-on.md`, `manifest.json` and `flat/SKILL.md` inside a plugin directory are read by the
installer; they are not copied into Claude Code.

## License

MIT — see [LICENSE](LICENSE).
