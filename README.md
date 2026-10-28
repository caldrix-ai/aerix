# aerix

Aerix is a coding agent for small local models. It works in a repository, reading files, editing them, running commands and handing pieces of the work to subagents. Every generation comes from a local `caldrix pool`; there is no hosted model. It reads the Claude Code configuration you already have and never writes to `~/.claude`.

## Install

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

`model` is anything `caldrix run` accepts. Set `--max-context` well above the startup context, as [tuning.md](docs/tuning.md) explains.

Then run Aerix in a repository, for the live screen or for one headless request:

```sh
aerix
aerix -p "explain what src/main.rs does"
```

The pool starts itself. When nothing listens on the pool socket and `pool.autostart` is on, which is the default, Aerix starts `caldrix pool` in the background and waits for it to listen, up to `pool.start_timeout_secs`. The pool keeps running after Aerix exits and is shared by every Aerix process on the machine. Stop it with `caldrix pool stop`.

## Runs in place

Aerix runs every command and every edit in your working tree, in the live screen and headless alike. It has no permission prompts, no checkpoints, no command guard and no undo. Each request ends with the diff against the state it started from. Run Aerix in a git repository, where you accept that.

## What it reads from Claude Code

Aerix reads the Claude Code configuration you already have: the `hooks`, `env`, `enabledPlugins`, `extraKnownMarketplaces` and `attribution` keys of its settings files, instruction files such as `CLAUDE.md` and `AGENTS.md`, skills, agents, plugins and project memory. It never writes to `~/.claude`. See [configuration](docs/configuration.md).

## Docs, for humans and agents

The [docs](docs/index.md) are plain Markdown, one task per page: [install](docs/install.md), [how it works](docs/how-it-works.md), [configuration](docs/configuration.md), [profiles](docs/profiles.md), [tuning](docs/tuning.md), [troubleshooting](docs/troubleshooting.md). Agents can start at [caldrix.ai/llms.txt](https://caldrix.ai/llms.txt).

## License

You may use Aerix free of charge, including for work, as one person on a single system; deploying it for many users needs a licence (see [LICENSE.md](LICENSE.md)).

## Bugs and security

File bugs as [issues](https://github.com/caldrix-ai/aerix/issues/new/choose) and include `aerix --version` and `caldrix --version` output. Report security issues privately: see [SECURITY.md](SECURITY.md).
