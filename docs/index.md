# Aerix quickstart

Aerix is a coding agent for small local models. It works in a repository, reading files, editing them, running commands and handing pieces of the work to subagents. Every generation comes from a local `caldrix pool`; there is no hosted model. It reads the Claude Code configuration you already have and never writes to `~/.claude`.

The first Aerix release is not published yet.

Install Aerix with one line, as [install.md](install.md) describes:

```sh
curl -fsSL https://caldrix.ai/install-aerix.sh | sh
```

## Start

Name a model once, in `~/.config/aerix/settings.json`. This is an example, not a recorded configuration:

```json
{
  "pool": {
    "members": [
      {
        "id": "main",
        "role": "main",
        "model": "qwen3-0.6b",
        "flags": ["--backend", "cpu", "--max-context", "8192"]
      }
    ]
  }
}
```

`model` is anything `caldrix run` accepts. Set `--max-context` well above the startup context, as [tuning.md](tuning.md) explains.

Then run Aerix in a repository, for the live screen or for one headless request:

```sh
aerix
aerix -p "explain what src/main.rs does"
```

The pool starts itself. When nothing listens on the pool socket and `pool.autostart` is on, which is the default, Aerix starts `caldrix pool` in the background and waits for it to listen, up to `pool.start_timeout_secs`. The pool keeps running after Aerix exits and is shared by every Aerix process on the machine. Stop it with `caldrix pool stop`.

## Runs in place

Aerix runs every command and every edit in your working tree, in the live screen and headless alike. It has no permission prompts, no checkpoints, no command guard and no undo. Each request ends with the diff against the state it started from. In the live screen, the first plan of a request waits for you to accept it, edit it or abort. Run Aerix in a git repository, where you accept that.

## Not supported

- Remote models: every member is a local `caldrix pool` member, with no keys and no provider adapters.
- `web_fetch` and `web_search`: the names are reserved and refuse every call.
- Workflows: the `workflows` settings key is accepted and unused.
- Resuming subagents: a resumed session rebuilds the root agent only.

## All docs

- [Install](install.md): the one-line installer, what it does with caldrix, platforms, checksums and uninstalling.
- [How it works](how-it-works.md): the pool, the turn loop, tools, context management and delegation.
- [Configuration](configuration.md): settings files and every key, flags, instruction files, skills, agents, plugins and hooks.
- [Profiles and tuning prompts](profiles.md): the per-project prompt profile and `aerix tune`.
- [Tuning for your hardware](tuning.md): choosing the model and pool flags, and shrinking the startup context.
- [Troubleshooting](troubleshooting.md): each refusal Aerix or its installer prints, with its cause and fix.
