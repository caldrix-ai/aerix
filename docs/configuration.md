# Configuring Aerix

Everything here is taken from the code: the keys and defaults, the flags and the interactive keys.
`aerix config show` prints the effective settings, one line per value, each with the file it came
from or `default`.

## Settings files and precedence

Aerix reads six files, highest precedence first:

| # | File | Kind |
|---|---|---|
| 1 | `<cwd>/.aerix/settings.local.json` | Aerix |
| 2 | `<cwd>/.aerix/settings.json` | Aerix |
| 3 | `$XDG_CONFIG_HOME/aerix/settings.json` (default `~/.config/aerix/settings.json`) | Aerix |
| 4 | `<cwd>/.claude/settings.local.json` | Claude Code |
| 5 | `<cwd>/.claude/settings.json` | Claude Code |
| 6 | `~/.claude/settings.json` | Claude Code |

`<cwd>` is the directory Aerix runs in (or `--cwd`), not the git root: project settings, skills
and agents are read from the current directory's `.aerix/` and `.claude/`.

**Claude Code files** contribute only these keys: `hooks`, `env`, `enabledPlugins`,
`extraKnownMarketplaces` and `attribution`. Every other key in them (`permissions`, `model`,
`statusLine`, `theme` and the rest) is ignored, with one stderr line per distinct key:
`aerix: settings: ignored key "<key>" (from <file>)`; the live screen shows each as a note in its startup block. The same happens for an
unknown top-level key in an Aerix file, including `safety`, which no longer exists.

**Merging.** Files are applied lowest first. An object merges leaf by leaf, so a project file that
sets `context.tool_caps.read` keeps the user file's other caps, and one that sets
`pool.max_connections` keeps the user's `pool.members`. `hooks` and `env` merge one level: an entry
(an event name, a variable) from a higher file replaces the lower file's entry. Anything that is not
an object (a number, a string, an array such as `pool.members`) is replaced whole.
`enabledPlugins` and `extraKnownMarketplaces` may be spelled in either case form
(`enabled_plugins`, `extra_known_marketplaces`).

**Errors.** A file that is not JSON, not an object, or has a value of the wrong type is an error
naming the key and the file, and Aerix does not start. Unknown fields inside a section (a typo such
as `pool.member`) are errors too.

## Settings keys

### `pool`

How Aerix finds or starts the pool and which members it binds.

| Key | Type | Default | Effect |
|---|---|---|---|
| `pool.socket` | path or null | `null` | The pool's socket. Precedence: `--pool-socket`, then `$AERIX_POOL_SOCKET`, then this key, then caldrix's rule: `$XDG_RUNTIME_DIR/caldrix/pool.sock`, else `<cache root>/run/pool.sock` with the cache root `$CALDRIX_CACHE_DIR`, else `$XDG_CACHE_HOME/caldrix`, else `~/.cache/caldrix`. |
| `pool.frames` | `"cbor"` or `"json"` | `"cbor"` | The protocol framing; `--frames` overrides it. An autostarted pool is started with the same `--frames`. |
| `pool.autostart` | bool | `true` | Start a pool when nothing listens on the socket. |
| `pool.caldrix` | string | `"caldrix"` | The caldrix program used to start the pool (looked up on `PATH`). |
| `pool.start_timeout_secs` | integer | `900` | How long to wait for an autostarted pool to listen (tried every 500 ms), and for each `MemberLoad` of a member the pool does not hold. |
| `pool.max_connections` | integer | `4` | The most generations in flight at once; each holds its own connection. Further generations wait for a connection. |
| `pool.pool_flags` | array of strings | `[]` | Flags for an autostarted `caldrix pool` itself, placed before the first `--load`: `--idle-timeout`, `--context-budget` (see [tuning](tuning.md)). A member's `flags` are serving flags and go after its `--load`. |
| `pool.members` | array | `[]` | The members, below. |

Each `pool.members` entry:

| Field | Type | Default | Effect |
|---|---|---|---|
| `id` | string | required | A name for the entry. Aerix does not use it to bind: members are matched to the pool's by `model`, and the pool names a member by its model reference. |
| `role` | `"main"` or `"decision"` | required | `main` runs every agent. `decision` routes requests; without one, `main` routes. |
| `model` | string | required | A caldrix model reference, as `caldrix run` takes it. |
| `flags` | array of strings | `[]` | caldrix serving flags for this member (`--backend`, `--max-context`, `--threads`, ...), passed after `--load <model>` or with `MemberLoad`. |

Binding rules: with no members configured, a pool that holds
exactly one member uses it as `main`. With members configured there must be one `main`; a member
the pool does not hold is loaded when everything the pool holds is configured; a pool holding a
model you did not configure is refused, naming both (stop it or configure it). A `decision` member
that fails to load is skipped with one stderr line (the live screen shows it as a note in its startup block) and
`main` routes.

### `pools`

| Key | Type | Default | Effect |
|---|---|---|---|
| `pools.<name>` | object, the shape of `pool` | none | `--pool <name>` replaces `pool` with this entry whole; its unset fields take `pool`'s defaults, not the values in your `pool`. |

### `delegation`

| Key | Type | Default | Effect |
|---|---|---|---|
| `delegation.depth` | integer | `3` | The deepest agent that may still spawn. The root is depth 0; an agent at depth `depth` has no `agent` tool and a spawn there is refused. |
| `delegation.retries` | integer | `2` | How many times a piece is re-run after its verifier says `fail`; a piece gets `retries + 1` implementer attempts. |
| `delegation.planner_tools` | array of tool names | `["repo_map", "agent", "plan", "context"]` | The tools the root may call when the request is routed `edit` or `plan`. |
| `delegation.max_turns` | integer | `40` | The turn budget of the root (unless `--max-turns`), of each implementer (unless its definition sets `maxTurns`), and of each verifier. |
| `delegation.parallel` | integer | `4` | The most `agent` calls of one turn that run at the same time (at least 1). |

### `context`

| Key | Type | Default | Effect |
|---|---|---|---|
| `context.soft_ratio` | number | `0.70` | Compaction starts when a context holds more than this share of the member's window. |
| `context.target_ratio` | number | `0.50` | Compaction stops once the context is at or below this share. |
| `context.max_new_tokens` | integer | `1024` | The most tokens one turn may generate; capped at a quarter of the window. |
| `context.tool_caps` | object, tool name to bytes | `read` 32768, `bash` 16384, `grep` 16384, `glob` 8192, `agent` 4096, `skill` 16384, `default` 8192 | A tool result longer than its cap is cut and the rest spooled. Tools without an entry use `default` (8192 if `default` is removed). Background job output uses the `bash` cap; a `/name` expansion uses `skill`. |
| `context.repo_map_max_entries` | integer | `2000` | In the `repo_map` tree, a directory with more direct entries than this is shown as one line with its file count. It is not a cap on the whole map; the map's result is capped by `tool_caps` (`default`). |
| `context.skills_chars` | integer | `1200` | The most characters the system prompt's skill list may take: every model-invocable skill with its description (up to 200 characters each) if that fits, else the names only, else the names that fit and a count, with a pointer to the `skill` tool's `list: true`. |

### `tui`

| Key | Type | Default | Effect |
|---|---|---|---|
| `tui.plain` | bool | `false` | Start interactive mode as plain lines. |
| `tui.thinking` | bool | `false` | Show the model's `thought` before each final (ctrl+o toggles it). |
| `tui.tree_open` | bool | `true` | Start with the agent tree expanded (ctrl+t toggles it). |

### Keys shared with Claude Code

| Key | Type | Default | Effect |
|---|---|---|---|
| `hooks` | object | `{}` | Hook configuration, Claude Code's format. See [Hooks](#hooks): each file's hooks are loaded from the file itself. |
| `env` | object, string to string | `{}` | Extra environment variables for every shell the tools start (`bash`, foreground and background) and for hook processes, as Claude Code's `env`. They are not set for the pool. |
| `enabledPlugins` | object, plugin id to bool | `{}` | A plugin is loaded only when its `name@marketplace` id is `true` here. |
| `extraKnownMarketplaces` | object | `{}` | Extra marketplaces for `aerix plugin`; only entries whose `source` is `{"source": "directory", "path": ...}` are used. |
| `attribution` | any | `null` | Read and shown by `config show`; nothing uses it. |

### `tune`

What `aerix tune` and `aerix profile sync` reflect with (see [Profiles](profiles.md)).

| Key | Type | Default | Effect |
|---|---|---|---|
| `tune.member` | string or null | `null` | The reflection member: a caldrix model reference. `aerix tune` and `aerix profile sync` use it if the pool holds it, else load it with `MemberLoad` and unload it when they end; a pool that cannot load it is refused by name (exit 1). Unset: the main member reflects. Ordinary sessions never load it. |
| `tune.flags` | array of strings | `[]` | caldrix serving flags for loading `tune.member`, as a member's `flags`. |
| `tune.sessions` | integer | `20` | How many of the newest matching sessions `aerix tune` reads when `--sessions` is not given. |

### Others

| Key | Type | Default | Effect |
|---|---|---|---|
| `profile` | string or null | `null` | The profile a session attaches to; `null` is the project's own (named by the project key). `--profile` overrides it; a resumed session keeps the profile it recorded. |
| `workflows` | any | `{}` | Accepted for the workflows feature, which is not built; nothing reads it. |

## Command line

```
aerix [--plain] [--resume ID | --continue] [options]   interactive mode
aerix -p PROMPT [options]                               one headless request
aerix [options] < FILE                                  the same, the prompt from stdin
```

| Flag | Effect |
|---|---|
| `-p`, `--print PROMPT` | Run one request headless and exit; `-` reads the prompt from stdin. |
| `--output-format text\|json\|stream-json` | Headless output. `text` (default): the final text, a blank line, the diff of what the request changed. `json`: one object `{session, outcome, reason, result, diff, files_touched, turns, usage}`. `stream-json`: every session event as a JSON line, then the result object with `"kind":"result"`. |
| `--max-turns N` | The root agent's turn budget (headless only). |
| `--pool NAME` | Use `pools.NAME` instead of `pool`. |
| `--resume ID` | Continue session `ID` with this prompt. |
| `--continue` | Continue this project's latest session. Exclusive with `--resume`. |
| `--in-place` | Accepted and has no effect: headless always runs in place. |
| `--pool-socket PATH` | The pool's socket. |
| `--frames cbor\|json` | The protocol framing. |
| `--cwd DIR` | The project directory (default: the current directory). |
| `--plain` | Interactive mode as plain lines; a usage error with `-p`, `--output-format`, `--max-turns` or `--in-place`. |
| `--profile NAME` | The profile to attach (else `profile`, else the project's). A resumed session keeps the profile it recorded; a different `--profile` on a resume is a usage error naming both. |
| `--profile-refresh` | With `--resume` or `--continue` only: move the session's profile branch to its model base's head before the run, so it takes the base's tuning since the session started. |
| `--protocol-version` | Print the pool protocol version this aerix speaks and exit; `caldrix pool --protocol-version` prints the pool's. The two must match. |
| `--version`, `--help` | |

Mode selection: `-p` is headless. Without `-p`, a terminal on stdin is interactive and
a pipe on stdin is a headless prompt. `tui.plain`, `TERM` unset, empty or `dumb`, or a non-terminal
on either side gives plain mode. `NO_COLOR` removes colour.

Subcommands:

| Command | Effect |
|---|---|
| `aerix sessions list [--json]` | This project's sessions: id, start, origin, outcome, title. |
| `aerix sessions rm ID` | Delete a session: its log, spool files, index entry, and its `session/ID` branch in its profile. |
| `aerix tune [--profile NAME] [--model ID] [--sessions N] [--dry-run]` | Reflect on the profile's newest `N` sessions (default `tune.sessions`) of one model (default: the main member's) and commit the revisions on its model base, after a sync of that base; `--dry-run` prints each revision as a diff and commits none. See [Profiles](profiles.md). |
| `aerix profile [--profile NAME] list` | The profile's branches: name, branch, commit. |
| `aerix profile show BRANCH [FILE]` | The files on a branch, or one file's text. |
| `aerix profile log BRANCH` | The branch's history: commit, date, subject (at most 200). |
| `aerix profile diff A B` | The diff between two branches or commits. |
| `aerix profile edit BRANCH FILE` | Open the file in `$VISUAL`, else `$EDITOR`, else `vi`, and commit it when the guards pass. `main`, `session/` branches, `upstream.json`, `upstream/` and an overlay path of no loaded definition are not edited. |
| `aerix profile sync [--model ID]` | Reconcile a model base (default: every `model/*` branch) with the upstream that changed under it. |
| `aerix config show` | The merged settings, each value with its source. |
| `aerix plugin marketplace add SPEC` | Add a marketplace: `owner/repo`, a git URL (with `#ref`), or a local directory (`./`, `/` or `~/`). |
| `aerix plugin marketplace update NAME` | Pull one of Aerix's cloned marketplaces. |
| `aerix plugin install NAME@MARKETPLACE` | Install a plugin (`MARKETPLACE/NAME` is accepted). |
| `aerix plugin list` | Installed plugins: id, version, root, and `[claude]` for Claude Code's. |
| `aerix plugin update ID` | Re-install from the marketplace's current entry. |
| `aerix plugin remove ID` | Remove one of Aerix's installs; Claude Code's are never changed. |
| `aerix import claude [--project DIR \| --all]` | Import Claude Code sessions (default: this directory's `~/.claude/projects/` entry). |

Exit codes: `0` done; `1` failed or killed (the reason on stderr), or an error; `2` the turn budget
ran out; `3` no pool (none listening and none could be started, it holds other models, or it went
away during the run); `64` usage error.

## Instruction files and memory

Aerix puts these files, in full, into one pinned `instructions` segment at the start of every
context, each headed by its path. Lowest precedence first:

1. `~/.claude/CLAUDE.md`
2. `$XDG_CONFIG_HOME/aerix/AERIX.md`
3. For each directory from the git root (or the cwd outside a repository) down to the cwd:
   `CLAUDE.md`, `.claude/CLAUDE.md`, `AGENTS.md`, `AERIX.md`.
4. Memory: `MEMORY.md` from Claude Code's project memory directory
   (`~/.claude/projects/<cwd with every non-alphanumeric character as ->/memory/`) and from Aerix's
   (`$XDG_DATA_HOME/aerix/memory/<project key>/`).

A file reached twice (an `AGENTS.md` symlinked to `CLAUDE.md`) is read once. `@path` imports are
not expanded, and only `MEMORY.md` is read from a memory directory, not the files it lists. The
system prompt names Aerix's memory directory so the model can write there with its tools; Aerix
itself writes nothing to it.

## Skills

A skill is a directory holding `SKILL.md` with frontmatter. Aerix looks in `<cwd>/.aerix/skills/`,
`<cwd>/.claude/skills/`, `~/.claude/skills/`, then each enabled plugin's skill directories and its
`commands/*.md` files (one skill per file, named by the file). The first skill found with a name
wins.

| Frontmatter | Default | Effect |
|---|---|---|
| `name` | the directory (or file) name | The skill's name. |
| `description` | required | Shown in the system prompt's skill list; a skill without one is reported and skipped. |
| `user-invocable` | `true` | `false`: `/name` refuses it. |
| `disable-model-invocation` | `false` | `true`: the model cannot load it and it is not listed. |
| `allowed-tools` | none | Parsed; not enforced. |

In the body, `$ARGUMENTS` is replaced by the arguments and, for a plugin skill,
`${CLAUDE_PLUGIN_ROOT}` by the plugin's root. A prompt whose first line is `/name args` is replaced
by the skill's text before routing (capped under `tool_caps.skill`); the model loads a skill with
the `skill` tool. A file over 1 MiB, or not a regular file, is refused unread.

## Agent definitions

`*.md` files with frontmatter in `<cwd>/.aerix/agents/`, `<cwd>/.claude/agents/`,
`~/.claude/agents/` and enabled plugins' agent directories; the first with a name wins. The `agent`
tool's description lists each by name and description (cut to 120 characters).

| Frontmatter | Effect |
|---|---|
| `name` | Default: the file name. |
| `description` | Required. |
| `tools` | Claude Code tool names, mapped to Aerix's (`Bash(git:*)` is `bash`; `Task` is `agent`); `WebFetch`/`WebSearch` are skipped; unknown names are dropped with a note in the result. None listed: every tool. |
| `model` | `inherit` (or absent): the caller's member. `sonnet`, `opus`, `haiku`: `inherit`, with a note. Anything else: a pool member id (its model reference) or `main`/`decision`; an unknown one uses the caller's member, with a note. |
| `maxTurns` | A positive integer: the implementer's turn budget. |

The body is added to the implementer's context as a pinned `# Agent: <name>` segment.

## Plugins and marketplaces

Aerix installs Claude Code plugins. A marketplace is a git repository or a local directory with
`.claude-plugin/marketplace.json`. Plugin sources `github`, `url`, `git-subdir` and relative paths
are supported; `npm`, `archive` and `command` are refused by name. MCP declarations
(`.mcp.json`, `mcpServers`) are skipped with one line. Aerix's installs and clones live under
`$XDG_DATA_HOME/aerix/plugins/`; plugins Claude Code installed (`~/.claude/plugins/`) are used in
place. Installing does not enable: set `"enabledPlugins": {"name@marketplace": true}` in any
settings file. A plugin contributes skills, commands (as skills), agents and hooks.

## Hooks

Aerix runs Claude Code's hooks contract. Hooks come from the `hooks` object of every settings file
above (all six; a settings handler identical to one in a lower file runs once) and from each enabled
plugin's `hooks/hooks.json` and declared hooks. `config show` prints the merged `hooks` value, but
every file's hooks run.

Events fired: `SessionStart` (matcher source `startup` or `resume`), `UserPromptSubmit`,
`PreToolUse`, `PostToolUse`, `Stop`, `SubagentStart`, `SubagentStop`, `PreCompact` (trigger
`auto`), `SessionEnd`. Aerix's own events, same shape: `RouteDecision`, `AgentSpawn`, `Verify`,
`ContextCompact`. Matchers see Claude Code's tool names: `Read`, `Write`, `Edit`, `Bash`, `Glob`,
`Grep`, `Agent`, and Aerix's `RepoMap`, `Plan`, `Context`, `Skill`; inputs use Claude's field names
(`file_path`, `old_string`, `run_in_background`, ...).

A `command` handler runs under `sh` with the event JSON on stdin; exit 0 passes, exit 2 blocks with
stderr given to the model, other exits warn; `decision`, `reason`, `continue`,
`additionalContext`, `systemMessage` and updated input or output are honoured. The default timeout
is 600 s (30 s for `UserPromptSubmit` and `SessionEnd`). The environment carries
`CLAUDE_PROJECT_DIR` and `AERIX_PROJECT_DIR` (the git root, else the cwd), `AERIX_SESSION_ID`,
`settings.env`, and for a plugin's handlers `CLAUDE_PLUGIN_ROOT`, `CLAUDE_PLUGIN_DATA` and their
`AERIX_` twins. `transcript_path` is a Claude Code format projection of the session.

Where Aerix differs from Claude Code:

- `permission_mode` is `bypassPermissions`; `ask` and `defer` let the call go ahead with a system
  message, and a `defer`'s `updatedInput` is ignored (17). `PermissionRequest` hooks are reported
  and never run.
- Only `command` handlers run; `http`, `mcp_tool`, `prompt` and `agent` are noted once and skipped.
  An `if` rule of the `Tool(glob)` form is evaluated on tool events; other forms are skipped with a
  warning (18).
- A Stop or SubagentStop hook can keep an agent going at most 5 times per run (20).
- `continue: false` ends the agent that fired it: the root fails, a subagent is killed (25).
- Settings and their hooks come from the cwd; `CLAUDE_PROJECT_DIR` and the hook's working
  directory are the git root (26).
- `async` handlers run under their timeout and are ended when the run ends; their output reaches
  the model at the firing agent's next fire point; `asyncRewake` runs as `async` (30).
- `tool_response` is shaped per tool; a failed call fires `PostToolUse` with the error text, not
  `PostToolUseFailure` (31).
- Hooks never inherit `CLAUDE_PLUGIN_ROOT`, `CLAUDE_PLUGIN_DATA`, their `AERIX_` twins,
  `CURSOR_PLUGIN_ROOT` or `COPILOT_CLI` from Aerix's own environment (33).
- `disableAllHooks` is not read (34).
- `Notification` fires only as `idle_prompt` (the matcher value), once, after the interactive
  screen has waited 60 s on you after a request, with the message `Aerix is waiting for your
  input`; there are no permission prompts, so no other type fires. `Escalation` never
  fires: escalation is not built. `SessionEnd` gets 30 s, not Claude Code's 1.5 s.

## Keys and slash commands (interactive mode)

From `tui::HELP`:

| Key or command | Effect |
|---|---|
| enter | send |
| `\` then enter | new line |
| esc | cancel the running request, else clear the draft |
| plan review | on an empty draft: enter accept, `e` edit, esc abort; typing edits the draft |
| ctrl+c | clear the draft; twice within two seconds exit (cancelling a running request first) |
| ctrl+d | exit when nothing is typed |
| ctrl+o | show or hide thinking |
| ctrl+t | collapse or expand the agent tree |
| ctrl+l | redraw |
| up/down | walk earlier prompts |
| `?` or `/help` | print this list |
| `/diff` | show the session's diff now |
| `/plan` | show the current plan |
| `/quit` | exit |
| `/name` | any other `/name` is a skill |

## Where data lives

| Path | Contents |
|---|---|
| `$XDG_CONFIG_HOME/aerix/` (`~/.config/aerix/`) | `settings.json`, `AERIX.md` |
| `$XDG_DATA_HOME/aerix/sessions/<project key>/` (`~/.local/share/aerix/...`) | `<id>.jsonl` (the session), `<id>.claude.jsonl` (the Claude-format projection, a cache), `index.json`, `spool/<id>-<agent>-<call>.txt` (full tool results that were cut) |
| `$XDG_DATA_HOME/aerix/memory/<project key>/` | Aerix's project memory (`MEMORY.md` is read) |
| `$XDG_DATA_HOME/aerix/profiles/<name>.git` | A profile: a bare git repository of prompts and overlays per branch ([Profiles](profiles.md)) |
| `$XDG_DATA_HOME/aerix/plugins/` | `installed.json`, `marketplaces.json`, `marketplaces/`, `<id>/<version>/`, `data/<id>/` |
| `$XDG_STATE_HOME/aerix/pool-start.log` | An autostarted pool's stdout and stderr |
| `$XDG_STATE_HOME/caldrix/pool.log` | The pool's request log: token counts only, never text |

The project key is the first 16 hex digits of sha256 over the git remote `origin` (normalised to
`host/path`), else over the absolute path; two clones of one remote share sessions and memory.
`$XDG_CACHE_HOME/aerix/` is not used.
