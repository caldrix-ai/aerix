# Profiles and `aerix tune`

Prompts live in a per-project git profile with a branch per model, and `aerix tune` proposes revisions from your sessions.

## What a profile holds

The prompts are built into the binary: `system.md`, `routing.md`,
`brief.md` (the subagent brief; the placeholders `brief`, `check` and `done_looks_like`, each in double braces), `verifier.md`
(`brief`, `check`, `summary` and `files`, each in double braces), `tools/<tool>.md` for `read`, `write`,
`edit`, `bash`, `glob`, `grep`, `agent`, `context`, `plan`, `repo_map` and `skill`, and `tune/reflect.md`
and `tune/reconcile.md` (the prompts `aerix tune` and `aerix profile sync` use). A session does not
read them from the binary: it reads them from its **profile**, a bare git repository at
`$XDG_DATA_HOME/aerix/profiles/<name>.git` that Aerix creates and owns. The turn contract (the
`{"calls": ...}` / `{"final": ...}` shape and its example) is rendered by code into the system
segment and is in no profile.

- **The name.** By default the project key (one profile per repository); `profile` or `--profile
  NAME` attaches another. A name is `[A-Za-z0-9_-][A-Za-z0-9._-]*`; anything else is a usage error.
- **Branches.** `main` is the model-agnostic base; its first commit is the embedded set, laid out as the
  built-in prompts are. `model/<model>` is made from `main` the first time a session of the profile
  runs that model, the model being the main member's model reference exactly as configured,
  percent-encoded into one ref component (`model/qwen3-0%2E6b`); an alias and its full reference are
  two branches. The session reads its prompts once at startup from its model base's head (a resumed
  one: its recorded commit); `session/<id>` is made at that commit by the session's first request,
  which records `profile`, `profile_branch` and `profile_commit` in its `session.start`; a later
  commit does not change a running session.
- **Resume** reads the files the session tuned from the commit it recorded, even if the main
  member's model changed since; a file it never tuned follows the current release's embedded text
  (the rule below). `--profile-refresh` moves the session's branch to its model base's head first.
  A recorded commit that no longer exists is named in one line and the model base's head is used.
- **Nothing propagates upward**: no merge from a session branch to a model base, nor from a model
  base to `main`.
- **Startup lines.** A prompt the commit lacks comes from the embedded set, with a line naming it.
  An artifact whose recorded upstream changed (a newer Aerix's prompt, an edited skill file) is
  named stale, one line each; sessions never reconcile. A tuned one is used as it is. A built-in
  prompt never tuned (its text is still the upstream it records, as every file a model base
  inherited from `main` is until tuned) uses this release's embedded text, its line saying so, and
  nothing is written; `aerix profile sync` records it. A profile that cannot be opened is one line,
  and the session uses the embedded prompts. A `<cwd>/.aerix/prompts/` or
  `$XDG_CONFIG_HOME/aerix/prompts/` that still holds files (a symlink to a file counts, and a
  directory past the 256 entries the check looks at gets the line too) gets one line: they are
  ignored.
- **Skill and agent overlays.** A commit may hold `skills/<source>/<skill name>/SKILL.md` and
  `agents/<source>/<name>.md`, the source being the plugin id or `user`/`project`. A loaded skill or
  agent definition whose overlay path the commit holds takes the overlay's text instead of its
  file's. A name or source that is not one path component (empty, `.`, `..`, `.git`, holding `/`,
  `\` or a control character), or a plugin id `user` or `project`, has no overlay path: one line, and
  the definition loads from its file. Every prompt and overlay comes from the main member's branch.
  `project` overlays (`skills/project/...`, `agents/project/...`) are applied, tuned and synced only
  when the profile's name is the cwd's project key: another project attaching the profile with
  `--profile` has its own project skills at those paths, so there they are skipped, with one line
  naming the profile.
- **Upstream hashes.** `upstream.json` on a branch maps each tuned artifact to the SHA-256 of the
  upstream text it was tuned from; an overlay's upstream text is kept beside it as
  `upstream/<path>`. Staleness is that hash differing from the current upstream's.

## Editing by hand

Customizing by hand is editing a model branch: `aerix profile edit model/<model> FILE`, or git on
the bare repo. `edit` refuses `main`, a `session/<id>` branch (nothing reads a session branch's
later commits: a resumed session reads its tuned files from its recorded commit, and
`--profile-refresh` moves the branch to the model base, discarding them), `upstream.json` and `upstream/`, an overlay path no
loaded skill or agent has, and a text that drops what code depends on (below); it commits with a
compare-and-swap on the branch, so a concurrent write is refused by name, not lost.

- `aerix profile list` shows a profile's branches and `aerix profile show model/<model> system.md`
  one text. Sessions started after an edit use it; sessions already running, and resumed ones,
  keep their commit until `--profile-refresh`.
- Shorter system and tool text shrinks every context; a sharper `routing.md` changes which shape
  requests get. The turn format is not in any prompt: code renders it, so no edit can break the
  parser.

## `aerix tune`

`aerix tune` reads the profile's sessions whose `session.start` names this profile and the
model (newest first, at most `--sessions`; imported Claude Code sessions are left out), extracts
evidence per artifact (each request's outcome, failed verifier verdicts and their reasons, refused
and failed tool calls, dropped turns, compactions, a next request that reads as a correction,
skills and agent definitions used), and asks the reflection member once per artifact that has
evidence, under `context.max_new_tokens` and the `Json` constraint, for `{"revise": bool, "text":
..., "why": ...}`. Before committing, a revision is refused by name if it drops a placeholder of
`brief.md` or `verifier.md`, the verifier's verdict shape (`"verdict"`, `"pass"`, `"fail"`,
`"reason"`) or one of `routing.md`'s shape words (`answer`, `edit`, `investigate`, `plan`), or is
over twice its embedded size (an overlay: twice its source's); an empty or unchanged answer revises
nothing. The revisions are one commit on `model/<model>`, its message naming the reflection
member, the session ids and each artifact's `why`; nothing revised commits nothing and says so.
stdout carries one line per artifact asked (`revised`, `kept` or `refused`), then `committed
<commit> on <branch>` with a `review: aerix profile ... diff` line naming that commit against its
parent, or `nothing to revise (...)`. With `--dry-run` the revisions are not committed: after the
per-artifact lines, each revision is printed as `diff <path>` followed by its unified diff, then
`dry run: N revisions from M sessions; nothing is committed on <branch>`. When it has sessions to
read, tune first runs a sync of that base, dry run or not (the sync commits what it reconciles,
as `aerix profile sync` would); a path the sync leaves stale is not tuned.

Set `tune.member` to reflect with a different model than `main` (it is loaded for the run and
unloaded after) and `tune.flags` to its serving flags; `--sessions N` bounds how much history is
read. Every `revised` line carries the model's `why`, and the commit holds them all. Read the
commit before relying on the revisions, and revert by hand (`aerix profile edit`, or git on the
bare repo) what you do not want. `aerix tune --dry-run` prints each revision as a diff and
commits none, so you can read what the model would change first; the sync that precedes a tune
still commits what it reconciles.

Nothing measures whether a tuned prompt is better. A small model's revisions are a proposal to
read, not an improvement.

`aerix tune` commits what the model writes without a review step, and its evidence comes from your
sessions: requests, tool errors that quote repository files, verifier reasons. Text planted in any
of those reaches the reflection model, and a revision that carries it is committed on the model
base and used by every later session of that model: a prompt injection can persist in a profile.
This is a known limitation; no confirm step exists. `--dry-run` shows the revisions without
committing them; review what a run committed with the `review:` line it prints, `aerix profile
log model/<model>` and `aerix profile diff`, and revert with `aerix profile edit` or git on the
bare repo.

## `aerix profile sync`

`aerix profile sync` reconciles: for each tuned artifact whose upstream changed, the reflection
member gets the old upstream text, the diff to the new one and the tuned text, and answers
`{"text": ..., "why": ...}`; the guards above run on the answer, and the result is committed with
the new upstream hash and the `why`. An answer equal to the tuned text keeps it and records the new
hash. An untuned artifact simply takes the new upstream, and an embedded file the branch lacks is
added; neither asks the model. An artifact it cannot reconcile (its old text or its source is gone,
the answer is refused) is named and stays stale.

After an Aerix upgrade changes a built-in prompt, or after a skill or agent file you tuned changes,
startup names each stale artifact; `aerix profile sync` reconciles them (`aerix tune` syncs first).
A built-in prompt you never tuned already follows the new release at startup (its line says the
embedded text is used); sync records it.

`tune.member`'s load and unload are not leased: the pool has no lease, so running `aerix tune` or
`sync` while another session uses the same model as its reflection member can unload it under that
session.
