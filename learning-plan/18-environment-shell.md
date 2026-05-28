# 18 — Environment, PATH & Shell Config

## Goals
- Understand environment variables and how programs (and your web apps) read them.
- Master `PATH` — why "command not found" happens and how the shell finds programs.
- Customize your shell persistently with `.bashrc`/`.profile`, aliases, and a sane prompt.

## Concepts
- **Environment variables** are named values every process inherits from its parent. They configure behavior without changing code — which is why **12-factor web apps read config from the environment** (`DATABASE_URL`, `PORT`, `NODE_ENV`, secrets).
  - `printenv` or `env` — list all env vars. `echo $HOME` — print one. `printenv PATH`.
  - `VAR=value` — set for the current shell only. **`export VAR=value`** — set *and* pass to child processes (so the program you launch can see it). The difference matters: without `export`, your app won't inherit the variable.
  - `VAR=value command` — set a variable for just that one command's run (e.g. `PORT=8080 ./myapp`).
  - `unset VAR` — remove it.
- **Important built-in vars:** `HOME` (your home dir), `USER`, `PWD` (current dir), `SHELL`, `PATH` (below), `LANG`/`LC_*` (locale), `EDITOR` (default editor for tools like `crontab -e`), `TERM` (terminal type).
- **`PATH` — how the shell finds commands.** `PATH` is a colon-separated list of directories (`/usr/local/bin:/usr/bin:/bin:…`). When you type `nginx`, the shell searches these dirs **in order** and runs the first match. Consequences:
  - **"command not found"** = the program isn't installed *or* its directory isn't in `PATH`.
  - `which nginx` / `type nginx` / `command -v nginx` show *which* file would run.
  - Add a directory: `export PATH="$HOME/bin:$PATH"` (prepending makes your version win; appending makes it a fallback). Always include `$PATH` so you don't wipe the existing entries.
  - Running a program in the current dir needs `./prog` precisely because `.` is (correctly) **not** in `PATH` for security.
- **Where shell config lives (and when each loads) — the confusing part:**
  - **`~/.bashrc`** — runs for **interactive non-login** shells (a new terminal tab, `docker compose exec ... bash`). Put aliases, prompt, and interactive tweaks here. This is the file you'll edit most.
  - **`~/.profile`** (or `~/.bash_profile`) — runs for **login** shells (SSH login, `su - user`, console login). Put `PATH` and `export`ed environment here. On Debian, `~/.profile` typically sources `~/.bashrc` so interactive logins get both.
  - **System-wide:** `/etc/profile`, `/etc/bash.bashrc`, and `/etc/environment` (the last sets system-wide env vars, no scripting).
  - Apply changes without reopening the shell: `source ~/.bashrc` (or `. ~/.bashrc`).
- **Aliases & functions** — shortcuts you define in `.bashrc`:
  - `alias ll='ls -lah'`, `alias gs='git status'`, `alias ..='cd ..'`. List with `alias`; remove with `unalias`.
  - For anything needing arguments/logic, use a shell function instead: `mkcd() { mkdir -p "$1" && cd "$1"; }`.
- **The prompt (`PS1`)** controls what your prompt shows (user, host, path, git branch, colors). You can customize it in `.bashrc`. Even a small tweak — coloring root's prompt red — is a real safety feature.
- **Web/Docker relevance:** environment variables are the primary config mechanism for containers (`docker compose` `environment:` / `.env` files map straight onto these), and `PATH` issues are a top cause of "works in my shell, fails in cron/systemd/Docker" bugs because those run with a minimal environment.

## Exercises
1. Explore the environment: `printenv | sort | less`, then `echo "$HOME $USER $SHELL"`, then `printenv PATH | tr ':' '\n'` (one PATH dir per line — read the search order).
2. set vs export: `GREETING=hello`, then `echo $GREETING` (works) but `bash -c 'echo $GREETING'` (empty — child can't see it). Now `export GREETING` and repeat the `bash -c` — now it's visible. Explain why to Claude.
3. One-shot var: `FOO=bar printenv FOO` (set just for that command), then `printenv FOO` afterward (gone).
4. PATH demystified: make a script `mkdir -p ~/bin && printf '#!/bin/bash\necho "my hello"\n' > ~/bin/hello && chmod +x ~/bin/hello`. Run `hello` (fails — not in PATH). Then `export PATH="$HOME/bin:$PATH"` and run `hello` again (works). Confirm with `which hello`.
5. Make it permanent: add `export PATH="$HOME/bin:$PATH"` to `~/.bashrc`, `source ~/.bashrc`, open behavior in a fresh `bash` and confirm `hello` still works.
6. Aliases: add `alias ll='ls -lah'` and a `mkcd` function to `~/.bashrc`, `source` it, and use both. 
7. Safer prompt: ask Claude for a one-line `PS1` that colors the prompt red when you're root, add it to `~/.bashrc`, and see it apply.
8. Persistence reality check: edit `~/.bashrc` in the lab, then `docker compose down && up` and shell back in. Did it survive? (Depends on whether your home dir is on the persistent volume — discuss with Claude how to make shell config persist across rebuilds.)

## Best Practices & Pitfalls
- **Always include `$PATH` when extending it.** `export PATH="/new/dir:$PATH"` — writing `export PATH="/new/dir"` *erases* the system paths and breaks nearly every command in that shell. (If you do this by accident, open a fresh shell.)
- **`export` is the difference between "my shell sees it" and "my program sees it."** App config that isn't `export`ed won't reach the process you launch — a frequent "the app can't find its DATABASE_URL" bug.
- **Put env/PATH in `~/.profile` (login) and interactive bits in `~/.bashrc`.** Putting `export PATH` only in `.bashrc` means SSH/cron/systemd (non-interactive or login-only contexts) won't get it — a classic source of "works in my terminal, fails everywhere else."
- **`source` after editing, or open a new shell.** Editing `.bashrc` does nothing to the current shell until you `source ~/.bashrc`.
- **Pitfall: cron/systemd/Docker have a *minimal* environment.** They don't read your `.bashrc` and have a bare `PATH`. Scripts that work interactively fail there because a command isn't found or a var is unset — use absolute paths and set env explicitly (Steps 19–20).
- **Pitfall: secrets in shell history/config.** Don't hardcode passwords in `.bashrc` or type them as `export SECRET=...` (they land in `~/.bash_history`). Use `.env` files (Step 24) with `600` perms instead.

## Checklist
- [ ] I can set, export, and unset environment variables and explain `export`'s role.
- [ ] I understand `PATH` and can diagnose/fix a "command not found" via `which`/`type`.
- [ ] I can safely prepend a directory to `PATH` without clobbering it.
- [ ] I know which config runs for login vs interactive shells (`.profile` vs `.bashrc`).
- [ ] I can add aliases/functions and `source` my config to apply changes.

## Resources
- Bash startup files explained: https://www.gnu.org/software/bash/manual/bash.html#Bash-Startup-Files
- The Twelve-Factor App — config in the environment: https://12factor.net/config
- PS1 prompt generator: https://bash-prompt-generator.org/
- `man bash` → "PARAMETERS" and "Shell Variables"
