# How Aerix works

This page describes what the code does, so you can reason about what Aerix will do on your
machine.

## Two processes

```
 aerix (interactive or -p)                  caldrix pool (daemon)
 settings, sessions, hooks                  members: one engine thread each
 the loop: route, agents, tools   socket    contexts and a KV prefix store per member
 one connection per generation  ─────────►  constrained decoding, logprobs, the log
```

**The pool.** `caldrix pool` is a daemon on a Unix socket that holds one or more loaded models
(members). It loads a member the way `caldrix run` opens a model: the same model references,
download, precheck, fit and serving flags. Several members may be resident when the sum of their
memory footprints plus one context budget each fits the limit `caldrix run` uses; a member whose
footprint is unknown is only ever loaded alone. Each member runs on its own engine
thread with the single-stream loop (`--max-batch 1`), so generations on one member are served one
at a time, in queue order. The pool exits after `--idle-timeout` seconds (1800 by default) with no
connection open, or on `caldrix pool stop`.

**Members and roles.** Aerix binds two roles. `main` runs every agent: the root, implementers and
verifiers (an agent definition's `model` or the `agent` call's `member` can name another loaded
member). `decision` routes requests; when there is none, `main` routes.

**Autostart.** When nothing answers on the socket, Aerix starts
`<pool.caldrix> pool --socket <socket> --load <main model> <main flags>` in its own process group,
with output appended to `$XDG_STATE_HOME/aerix/pool-start.log`, and tries the socket every 500 ms
for up to `pool.start_timeout_secs`. The pool binds its socket only after every `--load` has
loaded, so this wait covers the model download and load. A configured `decision` member is then
loaded with a `MemberLoad` request. The pool is left running and every Aerix process on the
machine shares it.

## The protocol and resident prefixes

Aerix speaks `caldrix-protocol` version 2 (length-prefixed CBOR frames, or JSON with `--frames
json`). The unit is a **context**: a list of segments, each a piece of text with an id, a role
(rendered through the model's own chat template, so Aerix never writes role markers) and a pinned
flag. Aerix creates, edits (append, replace, remove), forks and drops contexts, and asks for a
generation on one. The daemon is the only source of token counts: after every change Aerix reads
each segment's count back, and nothing is estimated on the Aerix side.

A generation re-renders the context's tokens. Each member keeps a **prefix store**: KV snapshots
keyed by the token ids they hold, shared by all of that member's contexts, kept least recently used
first under `--context-budget` bytes (default a quarter of the memory limit, at most 8 GiB; `0`
keeps none). A generation finds the longest stored prefix of its tokens. The rows after it are filled
one by one through the decode path, which costs far more a row than the batched whole-prompt
prefill, so the pool resumes only when filling the rest that way is expected to cost less than
prefilling the whole prompt. It weighs the two with the rates it measured on that member, and
until a fill has been timed it assumes the row-by-row path is 8 times slower a row. A prompt
that shares only a chat header with what is stored is therefore prefilled whole. Every generation
stores its rows: an appended turn keeps its prompt and its output, and a probe that is not
appended, such as routing, keeps its prompt's rows, so routing's fixed instructions are computed
once. The daemon reports where a generation resumed as `recomputed_from`. An evicted prefix costs
a full prefill, never a refusal.

This is why a second request costs less to start: every context starts with the same system and
instruction segments, so a later request, a fork, a verifier or a new session in the same directory
finds that prefix stored. To see it on your machine, read the `segment` events of your root's
first generation and of a later one in your session log: the first computes the whole prompt, a
later one resumes near its end. A change to any text near the start (an instruction
file, the skill list, a prompt's text in the profile, the tool set) changes the tokens from that point, and
everything after it is computed again.

Before the first token of a long prefill the daemon sends `Progress` frames, which count as
activity for the client's 600 s wait between frames.

## A request, end to end

1. **Startup.** Aerix resolves the directory, settings, instruction files, enabled plugins, skills,
   agent definitions and hooks, opens or creates the session, then finds or starts the pool and
   binds the members. It then attaches the session's profile: it opens (or creates, seeding `main`
   from the embedded prompts) the profile repository, makes the main member's model base
   `model/<model>` from `main` if it is new, and reads the prompts and the skill and agent overlays
   from one commit, the model base's head for a new session, the recorded commit for a resumed one
   (a prompt the profile never tuned follows the release's embedded text instead).
   The session's first request makes `session/<id>` at that commit and records the commit in
   `session.start`. A profile
   that cannot be opened is one line, and the embedded prompts are used
   ([profiles](profiles.md)).
2. **Session hooks.** `SessionStart` fires (its context becomes a pinned `# Hook context` segment);
   a first line `/name args` is replaced by that skill's text; `UserPromptSubmit` fires (a block
   ends the request; its context is appended to the prompt).
3. **Routing.** The decision member classifies the request on a throwaway context holding only
   `routing.md` and the request: a generation of at most 8 tokens, constrained with `OneOf` to
   `answer`, `edit`, `investigate` or `plan`, with logprobs. An answer that matches none is `edit`.
   The choice and its scores are a `route` event and fire `RouteDecision`. Routing picks the shape
   only; the root always runs on `main`.
4. **The root agent `a1`.** Its context is the system segment, the instructions segment, any hook
   context, and the prompt. For `answer` and `investigate` it may call every tool. For `edit` and
   `plan` it is a planner: it may call only `delegation.planner_tools` (by default `repo_map`,
   `agent`, `plan`, `context`), and a note in the prompt tells it to delegate anything that reads
   files or runs commands. The system segment always lists the full tool set; the restriction is in
   the prompt note and the turn schema.
5. **The turn loop** runs until a final, a budget or a named failure (below).
6. **The end.** `session.end` is written, `SessionEnd` fires, and the diff of what the request
   changed is shown (interactive) or printed (headless): the working tree against a snapshot of it
   taken when the request started (`git stash create`, which touches nothing; `HEAD` when the
   tree was clean), so edits you had not committed before are not shown as the agent's.

In the live screen, the root's first `plan write` of each request waits for you: enter accepts, `e`
edits, esc aborts. Later plan writes, and subagents' plans, are shown, not gated.

## The turn

Every turn the model answers with one JSON object. The tools' descriptions are text in the system
segment; Aerix does not use the chat template's tool-call markup. The text that tells
the model this shape, with its example, is rendered by code as the `# Turns` section after
`system.md`, so no profile version can change it. When the member
supports the `Json` constraint, the generation is constrained to this schema, so its shape is
guaranteed:

```json
{"oneOf": [
  {"thought": "<string, at most 600>", "calls": [{"tool": "<name>", "input": {...}}]},
  {"thought": "<string, at most 600>", "final": "<string>"}
]}
```

`calls` is a `oneOf` over the callable tools, each with its own input schema. A verifier's `final`
is `{"verdict": "pass"|"fail", "reason": "<string>"}` instead. The `Json` constraint refuses
whitespace before and after the top-level value.

Parsing: a `thought` over 600 characters is cut; at most 8 calls are kept (the rest are dropped
with a note); each call gets an id `c1`, `c2`, ... A member without the `Json` constraint gets the
parse error back and is asked again, at most 2 more times. A turn that hit `max_new_tokens` before
its JSON closed is removed from the context and asked again shorter, within the same 2 retries.

The calls of a turn run in order on the agent's thread, except `agent` calls, which run after the
others, concurrently, up to `delegation.parallel` at a time. Each call goes through `PreToolUse`
first (a deny is the result `refused: <reason>`; updated input replaces the call's) and
`PostToolUse` after. Results are appended in call order as user segments headed `[<tool> <id>]`. A
tool error is a result (`error: ...`), never the end of the loop.

A final ends the agent, unless background jobs are still running (the agent waits for one) or a
`Stop`/`SubagentStop` hook asks it to continue.

## Tools

| Tool | Input | What it does |
|---|---|---|
| `read` | `path`, `offset`, `limit` (200) | Lines of a file with line numbers (each cut to 2000 characters); a directory gives its entries. |
| `write` | `path`, `content` | Creates or replaces a file; parent directories are created. |
| `edit` | `path`, `old`, `new`, `replace_all` | Replaces an exact string; refuses when `old` is missing, or found more than once without `replace_all`. |
| `bash` | `command`, `timeout_secs` (120, at most 600), `background` | Runs `sh -c` in the working directory in its own process group; stdout, then stderr, then `[exit N]`, `[signal N]`, `[timeout after Ns]` or `[cancelled]`. In the background it returns a job id at once; the output comes back as `[job jN exited N]` when it exits. |
| `glob` | `pattern`, `path` | Matching files, relative to the working directory (`*` crosses `/`). |
| `grep` | `pattern`, `path`, `glob`, `max_matches` (200) | Regex matches with file and line (files over 8 MiB skipped, lines cut to 400 characters). |
| `repo_map` | none | The file tree with sizes and languages, computed once per request. |
| `plan` | `op` (`read`, `write`), `text` | Reads or replaces this agent's plan; each write is a `plan` event. |
| `context` | `op`, `ids`, `text` | The context tool, below. |
| `agent` | `brief`, `check`, `definition`, `member` | Delegates a piece, below. |
| `skill` | `name`, `args`, `list` | Loads a skill's text (`$ARGUMENTS` replaced), or lists the skills. |
| `web_fetch`, `web_search` | | Reserved names; every call is refused. |

`glob`, `grep` and `repo_map` see the git file set (tracked plus untracked, not ignored) inside a
repository; elsewhere a walk that skips `.git`, `target` and `node_modules`. Paths may be absolute
or contain `..`; tools reach outside the working directory.

## Context management

The runtime, not the model, owns each context's budget. For a member with window `W`:
the soft limit is `soft_ratio × W` (0.70), the target `target_ratio × W` (0.50), and a turn asks
for `min(max_new_tokens, W / 4)` tokens.

- **Before every turn**, when the context is over the soft limit or has no room for the turn
  (`tokens + max_new_tokens` must stay under `W`), the runtime compacts: unpinned tool results
  oldest first, then unpinned assistant turns, each replaced by a marker such as
  `[compacted tool result of call c12 (read, 3210 tokens)]`, until the context is at or under the
  target. Segments of 24 tokens or fewer are skipped. `PreCompact` fires first (a block skips that
  pass) and `ContextCompact` after; the model is told what happened. System, instructions, briefs,
  user turns, summaries and the scratchpad are never compacted.
- **Still no room** after compacting is a hard-limit hit: the first cuts this turn's
  `max_new_tokens` to the room left (if that is at least 64 tokens); the second kills the agent with
  `context floor: no room ...`, and its parent sees the failure.
- **Tool output caps.** A result longer than its `context.tool_caps` entry is cut, and the whole
  original is written once to the session's spool with a pointer line:
  `[truncated: N more bytes; read with read(path="...", offset=1)]`. A result is then fitted to the
  room the window leaves (taking 3 bytes per token); if it still does not fit, the runtime compacts
  once, and failing that appends only the pointer.
- **The context tool** lets the model manage its own context: `status` lists segments with ids,
  labels, token counts and pinned flags; `drop` removes segments; `summarize` replaces segments with
  one pinned summary; `pin`/`unpin`; `note` appends to a pinned scratchpad that survives compaction.
  `drop` and `summarize` refuse the system, instructions and scratchpad segments. Every op is a
  `context.op` event.

## Delegation

An `agent` call runs a **piece**: an implementer, then a verifier, retried on failure.

- **Implementer.** It runs on a fork of the calling agent's shared prefix: the leading system and
  instruction segments only, so it sees none of its caller's conversation and its brief must say
  everything it needs. The prefix store resumes those rows. The definition's body
  (if named) and `brief.md`, filled with the brief and check, are appended. Its tools are the definition's (or all), without `agent` at the depth
  limit. Its id is `<caller>.<call id>`, such as `a1.c3`; retries are `a1.c3.r1`, ....
- **Verifier.** When the implementer ends `done`, a verifier (`a1.c3.v1`, ...) runs on a fresh
  context (the system segments and `verifier.md` with the brief, check, summary and files) with only
  `read`, `bash`, `grep`, `glob` and `repo_map`, and must answer a verdict. `Verify` fires with it.
  A verifier that ends without a verdict counts as `fail`.
- **Retries.** A `fail` reruns the implementer on a fresh fork of the caller's shared prefix with the
  failure reason appended, at most `delegation.retries` times; then the piece fails with
  `verification failed after N attempts`. An implementer that ends other than `done` (failed,
  killed, out of turns) ends the piece at once, without a retry.
- **The result** goes back to the caller as JSON `{summary, files_touched, checks_run, outcome}`,
  capped under `tool_caps.agent`.
- **Bounds.** An agent at depth `delegation.depth` cannot spawn; at most `delegation.parallel`
  pieces of one turn run at once; each agent has its turn budget.

`SubagentStart` fires before each implementer and verifier (agent type: the definition name or
`implementer`, and `verifier`); `SubagentStop` on each one's final; `AgentSpawn` with each spawn.

## Skills and hooks

Skills are listed in the system segment within `context.skills_chars` characters (see
[configuration](configuration.md#skills)); the model loads one with the `skill` tool, or you put
`/name args` on the first line of a prompt. Hooks are Claude Code's contract with the deviations
listed in [configuration](configuration.md#hooks). A hook's `continue: false` ends the agent that
fired it; a `Stop` hook may extend an agent at most 5 times.

## Sessions

A session is `$XDG_DATA_HOME/aerix/sessions/<project key>/<id>.jsonl`, one event per line:
`session.start`, `route`, `agent.spawn`, `segment` (each context segment with its token count),
`message` (each turn with its usage, each tool result), `context.op`, `compaction`, `plan`, `job`,
`hook`, `agent.end`, `session.end`. Every agent's events carry its id, so each context can be
rebuilt. `index.json` beside the sessions lists them (`aerix sessions list`).

- `--resume ID` and `--continue` (the latest session of the project) rebuild the root agent's
  context and plan from the log and append the new prompt. Subagents are not resumed.
  A resumed session reads its prompts from the profile commit it recorded, even when the main
  member's model has changed since; `--profile-refresh` moves its branch to the model base's head
  first.
- `aerix sessions rm ID` deletes a session's log, spool files and index entry, and its
  `session/ID` branch in its profile.
- Interactive mode runs each request as its own runtime over the session: from the second request on,
  the root context is rebuilt from the log, which the prefix store usually makes cheap.
- `<id>.claude.jsonl` is a Claude Code format projection, rebuilt from the session whenever a hook
  fires, for hooks that read `transcript_path`. The session is the record.
- `aerix import claude` copies Claude Code sessions into the same store tagged `claude-code`. They
  cannot be resumed.

## Tuning and syncing a profile

How profiles are kept and tuned is on the [profiles](profiles.md) page; this is what the code does.

A session commits nothing to its profile beyond `main`'s seed: at startup it creates the profile,
`main` and its model base when they are new, its first request makes its own `session/<id>`, and
when this release's
embedded set differs from `main`'s it commits that set on `main`, a commit naming the release. Commits come from `aerix tune`,
`aerix profile sync` and `aerix profile edit` (or git by hand), each one commit on a branch with a
compare-and-swap of the ref: a write that finds the branch moved retries from the new head, at most 3 times, then is
refused by name. Every git call is bounded by 30 s.

`aerix tune` binds the pool, collects the corpus (the newest `--sessions` Aerix sessions whose
`session.start` names the profile and the model), and loads the reflection member (`tune.member`,
else the main member). It syncs the model base first, then, for each artifact with evidence, makes
one generation on a throwaway context under `context.max_new_tokens` and the `Json` constraint.
Each answer goes through the guards, and the surviving revisions become one commit on
`model/<model>` (under `--dry-run`: a diff per revision on stdout, no commit). A revised path whose
text on the branch changed during the run (a hand edit) is refused, not overwritten. A `tune.member` the run loaded is unloaded when it ends; the pool has no
lease, so a session that uses the same model meanwhile can lose it. Sessions already running keep
the commit they started on; new sessions of that model start on the new head.

`aerix profile sync` asks the reflection member at most once per stale tuned artifact and commits
the reconciled texts with the new upstream hashes; with nothing tuned stale it binds no pool.

## The interactive screen

On a terminal, Aerix opens with a launch frame: the peak mark beside the AERIX wordmark, an info
line (the main model, its lane, the context window, the working directory with the home directory
written `~`, the version, and `resumed` on a resumed session) and a hints line that names the keys
and commands (the full list is the `?` help). The frame narrows with the terminal: the mark beside
the wordmark from 72 columns, the wordmark alone from 58, the word `aerix` below that.

Under the frame is the startup block. Its rows are `pool` (the main model, its lane, the CPU lane's
thread count when the pool knows it, the `decision` member if there is one, and the socket),
`profile` (the profile, its model branch, the session's commit and how many of the embedded prompts
that commit tunes, left out when the profile's `upstream.json` does not parse, or `embedded prompts`
when no profile could be opened), `loaded` (instruction
files, skills, agents, plugins and hooks) and `session` (its id, new or resumed). Every startup note
follows as one row with its level mark (ember `▲` for a warning, red `✗` for an error, plain text
for information), and `✓ ready in <time>` ends the block. Plain and headless runs print the same
notes to stderr as before, and their banner is still one line.

While a request runs, the live region shows a status line (the thinking icon, the elapsed seconds,
the root's turn against its budget, the tool calls of every agent, the queued prompts; `cancelling…`
after Esc) with a context line and a diff line under it. Each figure has one source:

- **Context line**: the pool's own figures for the root agent, never a client estimate. A root
  message's usage sets it, a running root generation moves it with the prompt tokens the pool
  reported plus the tokens received so far, and after a compaction it shows the client's count of
  the compacted context prefixed `≈` until the pool reports again. It is absent until one of these
  reports, so a freshly resumed session shows none until its first generation.
- **Diff line**: `git diff --numstat` against the commit the session started on, taken when each
  request finishes, shown as added lines, removed lines and files once a file differs; outside a
  git repository there is none. When a verifier agent has run, it ends with its state: pending
  while one runs, then the last one's verdict, passed or failed.
- **Stats line**: committed when a request finishes without an error: the completion tokens of the
  generations that streamed, over the time at least one of them was streaming (overlapping
  generations count once), the request's whole duration and the local time. It shows `–` instead of
  a rate when too little streamed to give one.

## Headless mode

`aerix -p` runs one request (new, or resumed with `--resume`/`--continue`) and exits. It runs in
place in your working tree, with no permission prompts, no checkpoint, no guard and no sandbox;
nothing it does can be undone by Aerix. `--in-place` is accepted and changes nothing.
The output ends with the diff of what the request changed (against the snapshot taken at the start), plus `untracked: <path>` for touched
files git does not know. Exit codes are in [configuration](configuration.md#command-line).

## Every loop's bound

| Loop | Bound |
|---|---|
| Turns of an agent | `delegation.max_turns` (40), `--max-turns` for the root, a definition's `maxTurns` |
| Re-asks for a turn that did not parse or was cut by the token limit | 2 (`TURN_RETRIES`) |
| Calls kept per turn | 8 |
| Hard-limit hits before an agent is killed | 2; a cut turn needs at least 64 tokens of room |
| Appends tried for one result | 3 (fitted, fitted after compaction, pointer alone) |
| Verifier retries of a piece | `delegation.retries` (2) |
| Delegation depth | `delegation.depth` (3) |
| Concurrent pieces per turn | `delegation.parallel` (4) |
| Concurrent generations | `pool.max_connections` (4) |
| Stop-hook continues | 5 per agent run |
| Routing generation | 8 tokens |
| Foreground `bash` | `timeout_secs` (120, at most 600), then TERM, KILL after 2 s; cancel checked every 100 ms |
| Waiting for background jobs at a final | the latest job's timeout plus 10 s, polled every 1 s; jobs still running when the agent ends are killed |
| Hook handlers | their `timeout` (600 s; 30 s for `UserPromptSubmit` and `SessionEnd`), then TERM, KILL |
| Pool handshake and control requests | 30 s |
| Between two frames of a generation | 600 s (`Progress` frames count) |
| Autostart | `pool.start_timeout_secs` (900), polled every 500 ms |
| Member load | `pool.start_timeout_secs` |
| git calls for the diff | 30 s; the remote lookup for the project key 10 s |
| git calls on a profile | 30 s each |
| Compare-and-swap retries of a profile write | 3 (`CAS_RETRIES`), then a named race refusal |
| Sessions `aerix tune` reads | `--sessions`, else `tune.sessions` (20) |
| Reflection calls | one per artifact for `tune`, one per stale tuned artifact for `sync`, each under `context.max_new_tokens` |
| The pool's idle exit | `--idle-timeout` (1800 s; 0 means never) |

Exhausting a bound is a named outcome: `budget` (exit 2), or `failed`/`killed` with the reason.
