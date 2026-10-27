# Install Aerix

One line installs Aerix, and caldrix when it is missing.

```sh
curl -fsSL https://caldrix.ai/install-aerix.sh | sh
```

Aerix goes to `~/.local/bin`. To install somewhere else, pass `--prefix`:

```sh
curl -fsSL https://caldrix.ai/install-aerix.sh | sh -s -- --prefix ~/bin
```

The installer never uses `sudo`. It prints the `PATH` line to add when the prefix is not on `PATH` yet, and then prints `aerix --version`. The next step is to name a model in `~/.config/aerix/settings.json`, as in the [quickstart](index.md#start).

## What it does with caldrix

Every generation comes from a local `caldrix pool`, so Aerix needs a `caldrix` that has one.

- When no `caldrix` is on `PATH`, the installer installs the latest Caldrix release into the same prefix, with Caldrix's own installer.
- When a `caldrix` is on `PATH`, the installer uses it as it is. If that `caldrix` has no `caldrix pool`, the installer refuses, and the message names the command that updates it.

The installer downloads and verifies the Aerix archive before it touches caldrix, so a missing archive or a bad checksum leaves nothing installed.

## The protocol check

Aerix and `caldrix pool` talk over a socket, and both print the version of that protocol. After copying `aerix` into the prefix, the installer runs `aerix --protocol-version` and `caldrix pool --protocol-version` and compares the two. When they differ it refuses and says which release to update; Aerix stays installed in the prefix. See [troubleshooting](troubleshooting.md#the-installer).

## Platforms

- Linux x86_64 with glibc 2.38 or newer.
- macOS 14.0 or newer on Apple silicon, the pool on Metal there.

Every other platform is refused, and so is a Linux host without glibc or with an older one.

## Checksums

Each release archive comes with a `.sha256` file. The installer checks the archive against it, then checks the unpacked files against the `SHA256SUMS` inside the archive. It uses `sha256sum`, or `shasum` on macOS, and refuses when neither is installed.

## Where data lives and uninstalling

The installer writes the `aerix` binary, and `caldrix` when it installed that, into the prefix and nowhere else. To uninstall, remove the binary:

```sh
rm ~/.local/bin/aerix
```

Aerix's settings, sessions, memory, profiles, plugins and logs live in the directories listed under [where data lives](configuration.md#where-data-lives); remove them as well to leave nothing behind.
