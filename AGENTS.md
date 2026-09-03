# AGENTS.md

## Purpose

`flatmap` maps a command over records read from stdin and writes one output record per input record, in order. It is `flatMap` for the shell.

```
stdin --> flatmap { split records ; for each record run <COMMAND> ; combine results } --> stdout
```

## Layout

| path         | role                                                 |
| ------------ | ---------------------------------------------------- |
| `index.html` | the flatmap shell script, served at `cli.flatmap.ai` |
| `install`    | the installer, served at `cli.flatmap.ai/install`.   |
| `page.html`  | landing page                                         |
| `.flatmap/`  | this repo's own scope, the default `./.flatmap`       |
| `CNAME`      | `cli.flatmap.ai`, via GitHub Pages                   |

The CLI is named `index.html` for the web: GitHub Pages serves it as the site's index, so `curl cli.flatmap.ai` returns the script while a browser is redirected to `page.html`. The installer is an extensionless file served at its own path, so `curl cli.flatmap.ai/install` returns it verbatim — it carries no redirect, since it is never served as HTML.

## Usage

No install, one-off:

```sh
bash <(curl -sL cli.flatmap.ai) [FLAG...] COMMAND [ARG...]
```

Installation:

```sh
bash <(curl -sL cli.flatmap.ai/install) [--path DIR]
```

Then:

`flatmap --help` (the `usage()` block in `index.html`) is the single source of truth for flags, environment and exit codes. Do not restate it here.

## Behavior

**Records.** Default: all of stdin is one record, no terminator added; empty stdin still runs COMMAND once, on an empty record. With `-z`: NUL-separated both ways, pairing with `find -print0`; an empty stream runs COMMAND zero times, like `xargs -r`. A terminal counts as an empty stream in both modes. A `-z` result must not itself contain a NUL — flatmap passes it through unchanged and a downstream reader will see a record boundary.

**Streaming.** One record at a time, piped to COMMAND as its stdin. COMMAND inherits flatmap's stdout directly, so bytes pass through verbatim, unbuffered, binary safe; `-z` only appends a NUL after each result, never captures it.

**Errors.** Sequential, fail-fast: on a non-zero COMMAND status flatmap stops at that record and exits that status. Earlier records are already on stdout, as is whatever the failing record wrote before giving up (unterminated under `-z`) — so a non-zero status means the output stream is truncated, not absent. Parallelism or a different error-handling strategy is COMMAND's own business.

**Diagnostics.** Usage errors exit 2; every other non-zero status is COMMAND's, propagated unchanged, so codes collide by design. Every flatmap diagnostic goes to stderr prefixed `flatmap:` — that prefix, not the exit code, is what distinguishes flatmap's errors from a command's.

## Extending

A command is an executable file at `$FLATMAP_SCOPE/commands/<name>`. flatmap execs it directly, so its `#!` line picks the interpreter — any language. The default scope is `./.flatmap`, so a project ships its own commands by adding `./.flatmap/commands/`, or by pointing `-s` / `$FLATMAP_SCOPE` elsewhere.

A command can call flatmap itself as `"$FLATMAP_BIN"` — to compose sub-invocations (race, allOf, allSettled, anyOf semantics, each an ordinary command) or to recurse into a sub-flatMap. Same CLI, same contract, nothing new to learn.

### Writing a command

- **Do** read the record from stdin, write the result to stdout, and write nothing else to stdout.
- **Do** send diagnostics to stderr prefixed `flatmap <name>:`.
- **Do** exit 0 on success; non-zero aborts the whole run.
- **Do** `chmod +x` the file — flatmap exits 2 without it.
- **Don't** assume stdin is non-empty, a terminal, or seekable.
- **Don't** exit ≥256. Statuses are mod 256, so `exit 256` reads as success.

## Editing index.html

- **Do** run `bash -n index.html` after every edit.
- **Do** keep it dependency-free: shell builtins and `cat`, nothing else.
- **Don't** add a flag where a command would do.
- **Don't** teach it to install itself — that is `install`, a separate script, so the CLI stays flag-light and dependency-free.

## Editing install

- **Do** run `sh -n install` after every edit, and `chmod +x` it.
- **Do** keep it POSIX sh: it has no bashisms and its `#!` line says so.
- **Do** keep installing a separate concern: it fetches `cli.flatmap.ai` over the network, so `curl`, `install` and `sudo` are fair game here and nowhere else.
