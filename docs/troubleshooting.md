# Troubleshooting

Each refusal Aerix or its installer prints, with its cause and fix.

Each quoted message is the constant part of what Aerix or its installer prints; the rest of the line (a path, a model, a number) varies.

## Exit codes

- `0`: the request finished.
- `1`: the request failed or was killed, with the reason on stderr. A refusal from a running pool also exits `1`.
- `2`: the turn budget ran out.
- `3`: no pool. None was listening and none could be started, the pool holds other models, or the pool went away during the run.
- `64`: usage error. Run `aerix --help`.

## No pool

### None is listening and autostart is off

> `and pool.autostart is off; start one with: `

**Cause.** Nothing listens on the pool socket and `pool.autostart` is false. The message ends with the command that would start the pool.

**Fix.** Run that command, or set `pool.autostart` to true.

### None is listening and no main member is configured

> `and pool.members has no main member to start one with`

**Cause.** Autostart is on, but `pool.members` has no member to load.

**Fix.** Add a member with `"role": "main"`, as in the [quickstart](index.md#start).

### The pool cannot be started

> `cannot start a pool at `

> `install caldrix or set pool.caldrix`

**Cause.** Running the `caldrix` command failed, usually because it is not on `PATH`.

**Fix.** Install Caldrix, or set `pool.caldrix` to its path. Caldrix is installed as described in the [Caldrix install doc](https://caldrix.ai/docs/install).

### The pool holds a model you did not configure

> `but pool.members wants `

**Cause.** A pool is already running with a model that is not in `pool.members`. The pool is shared by every Aerix process, so Aerix will not unload it.

**Fix.** Stop it with `caldrix pool stop`, or configure the model it holds.

### The pool holds several members and none is configured

> `configure pool.members to say which`

**Cause.** `pool.members` is empty and the running pool holds more than one member, so Aerix cannot tell which is main.

**Fix.** Configure `pool.members`.

### The pool went away

**Cause.** The pool exited or its connection failed during a run. The failure is printed with the prefix `pool: ` and the run exits `3`. A pool that answers with a refusal exits `1` instead.

**Fix.** Run the request again; Aerix starts a pool when none is listening.

## An autostart that fails

> `did not listen within pool.start_timeout_secs`

> `before listening`

**Cause.** The pool Aerix started did not listen within `pool.start_timeout_secs` (900 seconds by default), or it exited first. A large model loading on a slow lane can take longer than the default.

**Fix.** Read `$XDG_STATE_HOME/aerix/pool-start.log`, where the pool's output goes; the message names the file. Raise `pool.start_timeout_secs` if the model was still loading.

## A settings file

> `is not a valid settings file`

**Cause.** A settings file is not JSON, or its top level is not an object. The message names the file and the parser's reason.

**Fix.** Correct the file.

A key whose value does not fit, or a field a key does not have, is refused with the key and the file named. A top-level key Aerix does not know is not an error: it is ignored, and the startup block says so.

> `ignored key`

## The context floor

> `context floor: no room for `

**Cause.** After compaction the context still has no room for what the next step needs, and the agent that hit it ends. Its parent sees the named failure.

**Fix.** Raise `--max-context` in the member's `flags`, or shrink the startup context. See [tuning.md](tuning.md).

## The installer

The installer, `install-aerix.sh`, prints each refusal on one line to stderr and exits `1`. See [install.md](install.md).

### An option or platform it refuses

> `unknown option `

> `(the only option is --prefix DIR)`

> `--prefix needs a directory`

> `is not a supported platform; aerix ships for Linux x86_64 and macOS on Apple silicon`

**Cause.** The only option is `--prefix DIR`, and the platforms are Linux x86_64 and macOS on Apple silicon.

**Fix.** Pass only `--prefix DIR`, or install on a supported platform.

### A host below the floor

> `this host does not appear to use glibc (ldd --version gave no glibc version)`

> ` is below the floor `

> `below the macOS `

> `could not read this Mac's macOS version (sw_vers -productVersion)`

**Cause.** Linux needs glibc 2.38 or newer, and macOS needs 14.0 or newer. The installer could not read the version, or it is below the floor.

**Fix.** Update the host, or run Aerix on one that meets the floor.

### No release, or a download that fails

> `no aerix release is published yet; nothing was installed`

> `could not resolve the latest release version`

> `has no archive for `

> `could not download `

> `could not reach `

> `could not read `

**Cause.** No Aerix release is published, the release host could not be reached, a checksum file could not be read, the latest version could not be read, the release has no archive for this platform, or a download failed. Nothing is installed.

**Fix.** Try again later, or read the release list on GitHub at `https://github.com/caldrix-ai/aerix/releases`.

### A checksum that does not match

> `neither sha256sum nor shasum is installed, so the download cannot be verified`

> `the download's checksum does not match `

> `the archive's files do not match the checksums in its SHA256SUMS; nothing was installed`

**Cause.** Neither checksum tool is installed, or the download is damaged or was altered. Nothing is installed.

**Fix.** Install `sha256sum` or `shasum`, or run the installer again.

### A caldrix without a pool

> `, which aerix needs; update it with curl -fsSL https://caldrix.ai/install.sh | sh`

**Cause.** The `caldrix` on `PATH` has no `caldrix pool`. The Aerix archive was verified but nothing was installed.

**Fix.** Run the command the message names to update Caldrix, then run the Aerix installer again.

### Installing caldrix failed

> `the caldrix installer failed (exit `

> `could not download the caldrix installer from `

> `is empty; nothing was installed`

> `the caldrix installer did not install a caldrix with `

> `; the caldrix it installed stays in `

**Cause.** No `caldrix` was on `PATH`, and installing it failed. The message says whether Aerix was installed; it was not, except where the message says the caldrix stays.

**Fix.** Install Caldrix as described in the [Caldrix install doc](https://caldrix.ai/docs/install), then run the Aerix installer again.

### The protocol check fails

> `did not print its pool protocol (aerix --protocol-version)`

> ` speaks pool protocol `

> `install releases that match: curl -fsSL https://caldrix.ai/install.sh | sh updates caldrix, and https://github.com/caldrix-ai/aerix/releases lists aerix's; aerix stays installed in `

**Cause.** The installed `aerix` did not print its protocol, or its protocol differs from the one `caldrix pool` speaks. Aerix is left in the prefix.

**Fix.** Update Caldrix with the command the message names, or install the Aerix release that matches the `caldrix` you have.
