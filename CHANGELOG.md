# Changelog

Every aerix release, newest first. Each release's notes on GitHub are its section here.

## [0.1.0] - 2026-10-26

The first release. Aerix is a coding agent for small local models: it works in your terminal, in
your repository, and every generation comes from a `caldrix pool` on your own machine.

### Added

- `aerix` (the live screen) and `aerix -p "<request>"` (one headless request), in a repository:
  reading files, editing them, running commands and handing pieces of the work to subagents, with
  each request ending in the diff against the state it started from.
- Release archives for Linux on x86_64 (glibc 2.38 or newer) and macOS 14.0 or newer on Apple
  silicon, installed with `curl -fsSL https://caldrix.ai/install-aerix.sh | sh`. The installer
  installs Caldrix too when no `caldrix` is on `PATH`.
- Aerix needs a `caldrix pool` that speaks pool protocol version 2, the version this release speaks:
  Caldrix 0.2.0 or newer. `aerix --protocol-version` and `caldrix pool --protocol-version` print the
  two; the installer compares them and refuses when they differ.
- Reads the Claude Code configuration you already have (hooks, skills, agents, plugins, instruction
  files and memory) and never writes to `~/.claude`.

### Where it has run

- The tests against a real `caldrix pool`, and one headless request, on Metal on an Apple M1 Max.
- The same tests, and headless requests, on the CPU lane on Linux. AMD GPUs have not been run with
  the pool.

### Known limitations

- Headless and live alike, Aerix runs every command and edit in place in your working tree, with no
  sandbox, no permission prompts, no checkpoints and no undo. Run it in a git repository.
- No hosted models: every member is a local `caldrix pool` member, with no keys and no provider
  adapters.
