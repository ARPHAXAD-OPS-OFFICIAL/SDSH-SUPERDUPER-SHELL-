# sdsh — Super Duper Shell

**Version:** 1.3
**Publisher:** ARPHA11 OPS
**Target platform:** Ubuntu / KDE Neon (Konsole terminal)
**License:** BSD 2-Clause "Simplified" License

A lightweight, custom Unix shell written in C that wraps common Linux commands
in short, memorable aliases while passing everything else straight through to
the underlying system via `fork()` / `execvp()` / `wait()`.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Requirements](#requirements)
- [Building](#building)
- [Usage](#usage)
  - [Interactive mode](#interactive-mode)
  - [Script mode](#script-mode)
- [Command Reference](#command-reference)
  - [File & Directory Operations](#file--directory-operations)
  - [System & App Management](#system--app-management)
  - [Privilege Escalation](#privilege-escalation)
  - [Built-in Commands](#built-in-commands)
- [Prompt](#prompt)
- [Scripting with `.sdsh` Files](#scripting-with-sdsh-files)
- [Signal Handling](#signal-handling)
- [Error Handling](#error-handling)
- [Architecture Notes](#architecture-notes)
- [Known Limitations](#known-limitations)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

`sdsh` ("Super Duper Shell") is a minimal command interpreter designed as a
friendlier, mnemonic layer on top of standard Linux tools. Instead of typing
`rm`, `mkdir`, or `sudo apt install`, users type short, descriptive words like
`nuke`, `newfolder`, or `superuser grab`. Anything that isn't a recognized
alias or built-in is passed straight through to the system via `execvp()`,
so `sdsh` behaves like a normal shell for everyday commands.

`sdsh` talks directly to the Linux kernel through `fork()`, `execvp()`, and
`wait()` — there is no dependency on an existing shell (e.g. `/bin/sh`) to
run commands.

## Features

- **Colorful ANSI prompt** — `[sdsh] user@machine:cwd$`, rendered with
  distinct colors for each segment (works in any VT100-compatible terminal,
  tuned for Konsole).
- **Command translation layer** — maps custom alias words to real binaries
  (see [Command Reference](#command-reference)).
- **Built-in commands** — `about`, `cls`, `cd`, `whereami`, `exit`, handled
  in-process (no fork required).
- **External command pass-through** — anything not recognized as a built-in
  or alias is executed via `execvp()`, so standard binaries on `$PATH` work
  unmodified.
- **Script execution** — run a sequence of `sdsh` commands from a file
  (`.sdsh` extension by convention) with `./sdsh script.sdsh`.
- **Signal handling** — `Ctrl+C` (`SIGINT`) clears the current input line and
  redraws the prompt instead of killing the shell; child processes still
  receive the default `SIGINT` behavior.
- **Graceful error messages** — unknown commands, failed `fork()`/`exec()`
  calls, and bad `cd` targets are reported clearly instead of crashing.
- **Comment support in scripts** — lines beginning with `#` are ignored.
- **`~/.sdshrc` config file** — automatically sourced at startup (interactive
  or script mode) before anything else runs; missing file is not an error;
  pass `--norc` to skip it.
- **Any script extension** — `sdsh script.sdsh`, `sdsh script.dsh`, and
  `sdsh script.sh` all work identically; the extension is a naming
  convention only, sdsh just reads the file line-by-line.
- **Tilde expansion in `cd`** — `cd ~` and `cd ~/path` work in both
  interactive input and scripts.

## Requirements

- Linux (developed and tested on Ubuntu / KDE Neon)
- POSIX-compliant environment (`_POSIX_C_SOURCE 200809L`)
- GCC (or any C compiler supporting C99 and POSIX headers)
- Standard C library with `<unistd.h>`, `<sys/wait.h>`, `<signal.h>`,
  `<pwd.h>` support

## Building

```bash
gcc -Wall -Wextra -o sdsh sdsh.c
```

This produces a single executable, `sdsh`, with no external dependencies
beyond the standard C library.

## Usage

### Interactive mode

Run `sdsh` with no arguments to start the interactive REPL:

```bash
./sdsh
```

You'll see a colored prompt:

```
[sdsh] alice@neon-box:/home/alice$
```

Type commands as you normally would in a shell. Press `Ctrl+D` (EOF) or run
`exit` to quit.

### Script mode

Pass a script file as the first argument to run its commands line-by-line
without an interactive prompt:

```bash
./sdsh script.sdsh
./sdsh script.dsh
./sdsh script.sh
```

Any filename or extension works — `sdsh` doesn't inspect it, it just opens
the file and reads lines. `.sdsh`/`.dsh`/`.sh` are naming conventions, not
requirements.

Script files:
- Are read one line at a time.
- Support blank lines and `#`-prefixed comment lines (both are skipped).
- Execute the same translation/built-in/external dispatch logic as
  interactive mode.
- Stop early if a line runs the `exit` built-in.
- A missing script file is a **fatal error** (`sdsh` prints a message and
  exits with status 1) — this is a script you explicitly asked to run.

### Command-line flags

| Flag      | Effect                                              |
|-----------|-------------------------------------------------------|
| `--norc`  | Skip auto-sourcing `~/.sdshrc` (works in either mode) |

`--norc` may appear before or after a script path, e.g. both
`sdsh --norc script.sh` and `sdsh script.sh --norc` are valid.

## Command Reference

### File & Directory Operations

| sdsh command                | Translates to | Description                     |
|------------------------------|----------------|----------------------------------|
| `list <path>`                | `ls`           | List directory contents          |
| `listdat`                    | `lsblk`        | List block devices                |
| `newfolder <dir>`            | `mkdir`        | Create a new directory            |
| `nuke <target>`              | `rm`           | Remove files or directories       |
| `clone <src> <dst>`          | `cp`           | Copy files                        |
| `shift <src> <dst>`          | `mv`           | Move or rename files              |
| `peek <file>`                | `cat`          | Print file contents               |
| `spark <file>`               | `touch`        | Create an empty file              |

### System & App Management

| sdsh command             | Translates to     | Description                    |
|----------------------------|--------------------|----------------------------------|
| `grab <package>`           | `apt install <package>` | Install a package          |
| `refresh`                  | `apt update`       | Refresh package lists            |
| `evolve`                   | `apt upgrade`      | Upgrade installed packages       |
| `revive`                   | `reboot`           | Reboot the machine                |
| `sleep`                    | `poweroff`         | Shut the machine down             |

### Privilege Escalation

| sdsh command                    | Translates to             |
|-----------------------------------|-----------------------------|
| `superuser <command>`             | `sudo <command>`            |
| `superuser <alias> [args...]`     | `sudo <translated-command> [args...]` |

`superuser` fully composes with the alias tables above, so compound forms
work as expected:

```
superuser whoami          →  sudo whoami
superuser nuke /etc/x     →  sudo rm /etc/x
superuser grab vim        →  sudo apt install vim
superuser refresh         →  sudo apt update
```

### Built-in Commands

These run inside the `sdsh` process itself (no `fork()`):

| Command      | Description                                                |
|--------------|--------------------------------------------------------------|
| `about`      | Prints ASCII banner, version, and full command reference     |
| `cls`        | Clears the terminal screen                                   |
| `cd [path]`  | Changes directory (defaults to `$HOME` if no path given)     |
| `whereami`   | Prints the current working directory (equivalent to `pwd`)   |
| `exit`       | Exits the shell                                               |

Any command not listed above and not matching a translation alias is passed
through unchanged to `execvp()`.

## Prompt

The prompt is rendered as:

```
[sdsh] <username>@<hostname>:<cwd>$
```

- **Username** is resolved via `getpwuid(getuid())`, falling back to
  `"unknown"` if unavailable.
- **Hostname** is truncated to its short form (everything before the first
  `.`), falling back to `"localhost"`.
- **Current working directory** is retrieved via `getcwd()`, falling back to
  `"?"` if unavailable.

Each segment is colored independently using ANSI escape codes for
readability in Konsole and other VT100-compatible terminals.

## Config File (`~/.sdshrc`)

If `~/.sdshrc` exists, `sdsh` automatically runs it — line-by-line, using the
exact same execution path as a script — **before** the interactive prompt
appears or a requested script starts running.

- Runs in both interactive and script-invocation modes.
- Missing `~/.sdshrc` is completely normal; nothing is printed.
- Any other open failure (e.g. permission denied) is reported to `stderr`
  but is **not fatal** — `sdsh` continues starting up.
- Uses the same syntax as any other `sdsh` script: one command per line,
  `#` comments, blank lines ignored.
- Runs relative to whatever directory `sdsh` was launched from — it's a
  startup script, not a directory-context loader.
- Skip it entirely with `sdsh --norc`.

Example `~/.sdshrc`:

```sdsh
# Personal sdsh startup file
about
cd ~/projects
whereami
```

## Scripting with Script Files

A script is simply a plain text file containing one `sdsh` command per line.
Any extension works — `.sdsh`, `.dsh`, `.sh`, or none at all:

```sdsh
# Example: setup.sdsh  (or setup.dsh, or setup.sh, or just "setup")
refresh
grab git
newfolder ~/projects
cd ~/projects
whereami
```

Run it with any of:

```bash
./sdsh setup.sdsh
./sdsh setup.dsh
./sdsh setup.sh
```

Comments (`#`) and blank lines are ignored. Execution stops if a line
triggers the `exit` built-in or the end of the file is reached. Note that
`~/.sdshrc` (if present) runs *before* the script itself, unless `--norc`
is passed.

## Signal Handling

- `sdsh` installs a custom handler for `SIGINT` (`Ctrl+C`) using
  `sigaction()` with `SA_RESTART`.
- Pressing `Ctrl+C` at the prompt clears the current line and redraws the
  prompt rather than terminating `sdsh`.
- Signal handling uses only async-signal-safe calls (`write()`, not
  `printf()`) inside the handler itself.
- Child processes spawned via `execvp()` restore the default `SIGINT`
  behavior before exec, so long-running external programs (e.g. `grep`,
  `htop`) can still be interrupted normally.

## Error Handling

`sdsh` reports errors to `stderr` in red text rather than crashing, including:

- Unknown/failed external commands (`command not found`, `exec failed`)
- Failed `fork()` calls (e.g. process table exhaustion)
- Failed `cd` targets (nonexistent or inaccessible directories)
- Failed `waitpid()` calls
- Unreadable script files

## Architecture Notes

**Startup sequence** (`main()`):

1. Install the `SIGINT` handler.
2. Parse arguments for `--norc` and an optional script path.
3. Unless `--norc` was given, call `sdsh_load_rc()`, which resolves
   `$HOME/.sdshrc` (falling back to `getpwuid()->pw_dir` if `$HOME` is
   unset) and runs it via `sdsh_run_script()` with `must_exist = 0`
   (missing file is not an error).
4. If a script path was given, run it via `sdsh_run_script()` with
   `must_exist = 1` (missing file *is* a fatal error). Otherwise, enter
   `sdsh_repl()`.

**Per-line command pipeline** (`sdsh_execute()`), used identically for
interactive input, `~/.sdshrc`, and script files:

1. **Trim** — strip leading/trailing whitespace; skip blank lines and
   `#`-comments.
2. **Tokenize** — split the line into a `NULL`-terminated `argv[]` array
   (whitespace-delimited, max 128 tokens per line).
3. **Translate** — apply alias tables:
   - `apt`-wrapper aliases (`grab`, `refresh`, `evolve`) shift `argv` to
     inject the correct `apt` sub-command.
   - `superuser` rewrites `argv[0]` to `sudo` and recursively translates the
     sub-command, including compound `apt`-wrapper forms.
   - All other custom words are translated 1-to-1 against a static lookup
     table; unrecognized commands pass through unchanged.
4. **Dispatch** — built-ins are checked first (`sdsh_run_builtin`); if the
   command isn't a built-in, it's executed externally via
   `fork()` + `execvp()` + `waitpid()` (`sdsh_run_external`). The `cd`
   built-in expands a leading `~` or `~/path` via `sdsh_expand_tilde()`
   before calling `chdir()`.

`sdsh_run_script()` backs both script execution and rc-file loading — the
only difference is the `must_exist` flag, which controls whether a missing
file is fatal (explicit script) or silently ignored (optional rc file).

Key compile-time constants (`sdsh.c`):

| Constant           | Value | Purpose                                |
|---------------------|-------|-------------------------------------------|
| `SDSH_VERSION`      | `1.3` | Version string shown in `about`        |
| `SDSH_MAX_ARGS`     | `128` | Max tokens per command line              |
| `SDSH_MAX_LINE`     | `4096`| Max characters per input line            |
| `SDSH_CWD_MAX`      | `1024`| Max path length shown in the prompt      |

## Known Limitations

- No support for shell pipelines (`|`), redirection (`>`, `<`), or command
  chaining (`&&`, `;`) — each line is treated as a single command.
- No command history or tab-completion.
- No environment variable expansion (e.g. `$MYVAR`) beyond what
  `execvp()`/`getenv()` provide natively; only `~`/`~/path` is expanded,
  and only for `cd`.
- Alias tables are static and require recompilation to extend.
- `~/.sdshrc` is a flat startup script, not a key/value settings file —
  there's no dedicated syntax for things like custom aliases or colors;
  it can only run `sdsh` commands.
- Designed and tested primarily for Ubuntu / KDE Neon; other distributions
  or terminals may render ANSI colors differently.

## Contributing

Issues and pull requests are welcome. Please keep contributions consistent
with the existing code style (POSIX C, `sdsh_`-prefixed static functions,
inline documentation comments) and test on Ubuntu or KDE Neon before
submitting.

## License

`sdsh` is released under the **BSD 2-Clause "Simplified" License**.

```
BSD 2-Clause License

Copyright (c) 2026, ARPHA11 OPS
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright notice,
   this list of conditions and the following disclaimer.

2. Redistributions in binary form must reproduce the above copyright notice,
   this list of conditions and the following disclaimer in the documentation
   and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDER AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE
LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
POSSIBILITY OF SUCH DAMAGE.
```

See [`LICENSE`](./LICENSE) for the full license text.

---

*Maintained by ARPHA11 OPS.*
