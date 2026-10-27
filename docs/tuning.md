# Tuning Aerix for your hardware

A request's time is spent in `caldrix pool` (prefill and generation) and in the tools and hooks
it runs. This page lists the levers the code gives you and how to see their effect. It
makes no speed or quality claims. Measure on your own machine with the logs described at the end.

## The model, the backend and the window

A member is a caldrix model reference plus caldrix serving flags, set in `pool.members[].flags`:

```json
{"id": "main", "role": "main", "model": "qwen3-0.6b",
 "flags": ["--backend", "cpu", "--max-context", "8192", "--threads", "8"]}
```

- **`model`**: anything `caldrix run` takes. `caldrix doctor` reports which lanes your machine has;
  `caldrix models supported` lists what caldrix runs. Quantized checkpoints are not supported.
- **`--backend auto|hip|metal|cpu`**: `auto` takes the first usable lane in the order Metal, HIP,
  CPU. `caldrix pool` has been run on the CPU lane and on Metal. Nothing has been run with the pool
  on HIP.
- **`--max-context N`** (`--ctx N` is the same): the member's window. caldrix's default is 2048
  tokens, which can be smaller than Aerix's startup context in a repository with long instruction
  files (see the next section for how to read yours). The window must hold the startup context, `max_new_tokens` and the results
  the agent reads; if the first context does not fit, the pool refuses it and the request fails. A
  larger window takes more KV memory, which the pool's fit counts.
- **Other serving flags** are inferno-serve's and are forwarded unchanged: `--threads` and
  `--cpu-isa` (CPU lane), `--dtype`, `--weight-budget`, `--queue-depth` (default 8) and the rest.
  Leave `--max-batch` alone: the pool passes `--max-batch 1` because constraints and logprobs need
  the single-stream loop, and a batched member advertises neither.

**Pool flags** come before the first `--load` on the `caldrix pool` command line:

| Flag | Default | Effect |
|---|---|---|
| `--socket PATH` | `$XDG_RUNTIME_DIR/caldrix/pool.sock`, else `<cache root>/run/pool.sock` | Where the pool listens. |
| `--idle-timeout SECS` | 1800 | Exit after this long with no connection; `0` never exits. |
| `--frames cbor\|json` | cbor | Framing; Aerix's `pool.frames` must match. |
| `--context-budget BYTES` | a quarter of the memory limit, at most 8 GiB | Each member's prefix store; `0` keeps nothing (every generation is a full prefill). |
| `--load MODEL [serving flags]` | | One member; repeat for several. |

An autostarted pool gets `--socket`, `--frames` from `pool.frames`, then `pool.pool_flags`, then
`--load <main> <main flags>`. Put pool flags in `pool.pool_flags`, for example
`"pool_flags": ["--idle-timeout", "0", "--context-budget", "4294967296"]`; a member's `flags` are
placed after its `--load`, so they are serving flags.

**Several members.** Give a second member `"role": "decision"` and Aerix loads it after `main`
(with its own flags); routing then runs on it instead of `main`. Or start the pool with two
`--load` groups. A load succeeds only when the members' footprints plus one context budget each fit
the memory limit; a member whose footprint is unknown (streamed weights) can only be alone. Whether
a smaller decision model makes routing cheaper on your machine is something to measure: routing is
a generation of at most 8 tokens on a context holding `routing.md` and the request.

## The startup context

Every context starts with the same two pinned segments, and the first request in a directory pays
a full prefill of them:

- **System**: `system.md` from the session's profile commit, then the code-rendered turn contract,
  then every tool's description and input fields (the full tool
  set, always), the `agent` tool's list of agent definitions (each description cut to 120
  characters, no total budget), the skill list (within `context.skills_chars`), and the project's
  memory path.
- **Instructions**: every instruction file and `MEMORY.md`, in full (see
  [configuration](configuration.md#instruction-files-and-memory)).
- A `SessionStart` hook's context, if any, as a third pinned segment; then the prompt.

The system segment grows with the number of tools, agent definitions and skills you have, and the
instructions segments with your instruction files. To see yours, read your root's `segment` events
in the session log:

```sh
jq -c 'select(.kind == "segment" and .agent == "a1") | {id, label, tokens}' SESSION.jsonl
```

To shrink it:

- **Instruction files** are included whole: `~/.claude/CLAUDE.md` in every project, then every
  `CLAUDE.md`, `.claude/CLAUDE.md`, `AGENTS.md` and `AERIX.md` from the git root down to the cwd,
  and both `MEMORY.md` files. Shorter files are the largest lever you control per project.
- **Skills**: lower `context.skills_chars` to list names only (or fewer names); set
  `disable-model-invocation: true` on skills the model should not see; or turn plugins off in
  `enabledPlugins`. Disabling a plugin also drops its agent definitions and hooks.
- **Agent definitions** have no budget: every definition found in `.aerix/agents/`,
  `.claude/agents/`, `~/.claude/agents/` and enabled plugins adds a line.
- **Prompts**: edit `system.md` or `tools/<name>.md` on your model's profile branch to shorter
  text (below).

`context.tool_caps` and the repo map do not change the startup context: they bound what later
tool results add. The `repo_map` result has no cap of its own and uses `tool_caps.default` (8192
bytes) unless you add a `repo_map` entry; `repo_map_max_entries` collapses large directories.

**Keep the prefix stable.** The prefix store resumes on identical leading tokens. Editing an
instruction file, a prompt on the profile branch, the skill set or the settings that shape the system segment
makes the next request recompute from the first changed token. The memory path at the end of the
system segment differs per project, so two projects share at most the text before it.

## Context ratios and `max_new_tokens`

- `context.max_new_tokens` (1024, at most a quarter of the window) is reserved on every turn. A
  lower value leaves more room for results; a turn that runs out before its JSON closes is dropped
  and asked again shorter, which costs another generation (at most 2 re-asks).
- `context.soft_ratio` (0.70) and `context.target_ratio` (0.50) decide when compaction starts and
  where it stops. Only unpinned tool results and assistant turns can be replaced, so if the pinned
  segments alone are near the target, compaction replaces everything it can and still stays above
  it. Size the window so the startup context sits well below `target_ratio × window`.
- Each compaction edits the context; the next generation recomputes from the first replaced
  segment, so frequent compaction also costs prefill.
- `context.tool_caps` bound each result in bytes; anything cut is in the spool and the model can page
  through it with `read`.

## Delegation

- Each piece is at least two agents: an implementer on a fork of the caller's shared prefix (system
  and instruction segments, resumed from the prefix store) with its brief, and a
  verifier on a fresh context (its system segments usually resume from the prefix store). A
  `fail` verdict reruns the implementer, up to `delegation.retries` (2) times.
- `delegation.parallel` (4) runs pieces concurrently, but one member serves one generation at a
  time: concurrent pieces overlap their tool work, while their generations queue on the engine.
  Keep `pool.max_connections` (4) at least as large, or generations wait for a connection.
- `delegation.depth` (3) and `delegation.max_turns` (40) bound how much work one request can
  spawn. Lower them to cap the cost of a request.
- `delegation.planner_tools` is what the root may call for `edit` and `plan` requests. The default
  leaves out `skill`; add it if the planner should load skills itself.

## Prompts

The prompts are the agent's tunable text and live in the project's profile. Shorter system and tool
text shrinks every context; a sharper `routing.md` changes which shape requests get. How to edit
them by hand or with `aerix tune` is on the [profiles](profiles.md) page.

## Keeping the pool warm

The model load and the first prefill happen once per pool, not once per request. The pool is
shared by every Aerix process on the machine, and its prefix store serves every session.

- An autostarted pool exits after 1800 s with no connection. To keep it up, set
  `"pool_flags": ["--idle-timeout", "0"]`, or start it yourself with the same model and flags as
  `pool.members`:

  ```sh
  caldrix pool --idle-timeout 0 --load qwen3-0.6b --backend cpu --max-context 8192
  ```

  Aerix connects to it if the socket is the same (default rule on both sides, or `--socket` with
  `pool.socket`). It refuses a pool that holds a model you did not configure.
- `caldrix pool stop` ends it. Changing a member's flags takes a new pool: Aerix binds by model
  reference and does not reload a member that is already resident.
- If an autostart fails, `$XDG_STATE_HOME/aerix/pool-start.log` has the pool's output.

## Seeing where the time goes

**The pool log**, `$XDG_STATE_HOME/caldrix/pool.log` (default `~/.local/state/caldrix/pool.log`),
has one JSON line per event and never any text: `ts` (milliseconds since the epoch, written when
the event ends), `event` (`member_load` or `generate`), `member`, `prompt_tokens`,
`completion_tokens`, `finish` and `recomputed_from` (the row the generation resumed at; 0 is a full
prefill).

```sh
jq -c '{ts, event, member, prompt_tokens, completion_tokens, recomputed_from, finish}' \
  ~/.local/state/caldrix/pool.log
```

A request shows as a routing line (a small prompt, a few completion tokens) followed by the root's
turns. On the first request in a directory the root line has `recomputed_from 0`; on a repeat
request it should be close to `prompt_tokens`. A `recomputed_from` far below `prompt_tokens` on a repeat request
means the prefix was not stored (it was evicted, `--context-budget` is 0, or the leading text
changed), or that the stored prefix was too short to pay for a row-by-row fill of the rest, and
the pool prefilled the whole prompt instead ([how it works](how-it-works.md)).

**The session log**, `$XDG_DATA_HOME/aerix/sessions/<project key>/<id>.jsonl` (the newest file in
that directory is the latest session), has a `ts` on every event, the usage of each turn, and every
route, compaction, hook and job:

```sh
jq -c 'select(.kind == "message" and .role == "assistant") | {ts, agent, usage}' SESSION.jsonl
jq -c 'select(.kind == "compaction") | {ts, agent, tokens_before, tokens_after}' SESSION.jsonl
jq -c 'select(.kind == "hook") | {ts, event, command, exit}' SESSION.jsonl
```

The gap between an assistant message and the event before it is that turn's generation time; long
gaps around `hook` or `job` events are hooks and commands, not the model. `aerix -p ... --output-format
json` also reports the run's total `turns` and `usage`.
