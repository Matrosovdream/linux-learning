# 19 — Bash Scripting

## Goals
- Write, make executable, and run a bash script.
- Use variables, arguments, conditionals, loops, and functions.
- Write *safe* scripts with `set -euo pipefail` and proper quoting.
- Automate a small real task (e.g. a backup or setup script).

## Concepts
- **Anatomy of a script:**
  - **Shebang:** the first line `#!/usr/bin/env bash` tells the kernel which interpreter to use. Make the file executable (`chmod +x script.sh`) and run it `./script.sh`, or run without `+x` via `bash script.sh`.
  - Comments start with `#`. Keep scripts in version control alongside your infra.
- **Variables & quoting (the part that bites everyone):**
  - Assign with **no spaces**: `name="web"` (not `name = "web"`). Use with `$name` or `${name}`.
  - **Always quote expansions:** `"$var"`, `"$@"`, `"${arr[@]}"`. Unquoted variables undergo word-splitting and globbing — the source of countless bugs (a path with a space becomes two arguments). Rule of thumb: quote every variable unless you have a specific reason not to.
  - **Command substitution:** `today="$(date +%F)"` captures a command's output into a variable.
  - **Arithmetic:** `count=$(( count + 1 ))`.
- **Script arguments:** `$1`, `$2`, … are positional args; `$0` is the script name; `$#` is the count; `"$@"` is all of them (quoted, each separate); `$?` is the last exit code. Example: `./deploy.sh prod` → `$1` is `prod`.
- **Conditionals:**
  - `if cmd; then … elif cmd; then … else … fi` — branches on a command's *exit code* (0 = true).
  - **`[[ ... ]]`** is bash's test: `[[ -f "$f" ]]` (file exists), `[[ -d "$d" ]]` (dir exists), `[[ -z "$s" ]]` (empty string), `[[ -n "$s" ]]` (non-empty), `[[ "$a" == "$b" ]]` (string equal), `[[ "$n" -gt 5 ]]` (numeric). Prefer `[[ ]]` over the older `[ ]` — it's safer with spaces and supports `&&`/`||`.
  - `case "$1" in start) …;; stop) …;; *) …;; esac` — clean multi-way branching (great for `start|stop|restart` scripts).
- **Loops:**
  - `for f in *.log; do echo "$f"; done` — iterate over words/files. `for i in {1..5}; do …; done` — ranges.
  - `while read -r line; do …; done < file` — read a file line by line (the `-r` prevents backslash mangling).
  - `until cmd; do …; done` — loop until a command succeeds (handy for "wait for the DB to be ready").
- **Functions:** `greet() { echo "hi $1"; }` then call `greet world`. Use `local var` inside functions to avoid clobbering globals. `return N` sets an exit code; `echo` "returns" data via stdout.
- **Safety: start every serious script with `set -euo pipefail`:**
  - `set -e` — exit immediately if any command fails (no silently continuing after an error).
  - `set -u` — error on use of an **unset** variable (catches typos like `$databse_url`).
  - `set -o pipefail` — a pipeline fails if *any* stage fails, not just the last.
  - Together they turn silent, dangerous failures into loud, early ones. Add `IFS=$'\n\t'` to tame word-splitting further.
- **Debugging:** `bash -x script.sh` traces every command as it runs (or `set -x` inside). **`shellcheck script.sh`** (`apt install shellcheck`) is a linter that catches quoting bugs, unset vars, and footguns before they bite — run it on everything.
- **Web/Docker relevance:** entrypoint scripts, deploy/backup scripts, "wait for dependency then start" scripts, and provisioning steps are all bash. The same safety habits prevent a deploy script from half-running and leaving a broken server.

## Exercises
1. Hello script: create `~/bin/hello.sh` with a shebang that prints a greeting; `chmod +x` it; run `./hello.sh` and `bash hello.sh`. Confirm the shebang line's role by removing `+x` and running each way.
2. Arguments: write `greet.sh` that prints `Hello, $1!` and errors with a usage message if `$# -eq 0`. Test `./greet.sh Stan` and `./greet.sh` (no arg).
3. Conditionals: write `checkfile.sh "$1"` that reports whether the path is a file, a directory, or doesn't exist (`[[ -f ]]`/`[[ -d ]]`/else). Test against `/etc/hostname`, `/etc`, `/nope`.
4. Loop over files: write a script that, for each `*.log` in a directory arg, prints the filename and its line count (`wc -l`). Create a few logs to test.
5. `while read` a file: loop over `/etc/passwd` lines, split on `:` (set `IFS=:` and `read -r user _ uid _`), and print users with UID ≥ 1000.
6. A real one — backup script: write `backup.sh DIR` that tars `DIR` into `/work/backups/DIR-YYYY-MM-DD.tgz` (use `date +%F`), creating the backup dir if needed, with `set -euo pipefail` at the top and a usage check. Run it on `/work/lab`.
7. Make it safe & clean: run `shellcheck backup.sh`, fix every warning, then `bash -x backup.sh /work/lab` and read the trace. Ask Claude to review the final script.

## Best Practices & Pitfalls
- **Start scripts with `set -euo pipefail`.** Without it, a failed `cd` or unset variable lets the script barrel on and do damage (the infamous `rm -rf "$DIR/"` where `$DIR` was never set). Fail fast, fail loud.
- **Quote everything: `"$var"`, `"$@"`.** Unquoted variables split on spaces and expand globs — the single most common bash bug. A filename with a space or a `*` will wreck an unquoted script.
- **Run `shellcheck` on every script.** It catches the quoting, unset-var, and exit-code mistakes you'll otherwise learn the hard way. Treat its warnings as errors.
- **Use `[[ ]]`, not `[ ]`,** and `$(( ))` for math. The old `[ ]` mishandles empty/spaced values; `[[ ]]` is bash-native and safer.
- **Pitfall: parsing `ls` output or filenames with loops.** Filenames can contain spaces and newlines. Use globs (`for f in *.txt`) or `find -print0 | while read -r -d ''` rather than `for f in $(ls)`.
- **Pitfall: scripts assuming an interactive environment.** When run by cron/systemd they have a bare `PATH` and no `.bashrc`. Use absolute paths or set `PATH` at the top, and don't rely on aliases (aliases don't work in scripts anyway).
- **Pitfall: `set -e` doesn't catch everything.** Failures in conditions, command substitutions, and some pipelines can slip past; that's why `pipefail` and explicit error checks still matter for critical steps.

## Checklist
- [ ] I can write a script with a shebang, make it executable, and run it.
- [ ] I use variables and arguments correctly and **quote** all expansions.
- [ ] I can write `if`/`case` conditionals and `for`/`while` loops.
- [ ] I start scripts with `set -euo pipefail` and understand what each flag prevents.
- [ ] I run `shellcheck` and can debug with `bash -x`.

## Resources
- Bash scripting guide (Shotts, Part 4): https://linuxcommand.org/tlcl.php
- ShellCheck (lint your scripts): https://www.shellcheck.net/
- Google Shell Style Guide: https://google.github.io/styleguide/shellguide.html
- Bash pitfalls (read this — it's gold): https://mywiki.wooledge.org/BashPitfalls
