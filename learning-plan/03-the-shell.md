# 03 — The Shell & Command Line Basics

## Goals
- Understand the anatomy of a command: program, options, arguments.
- Get help on any command three ways: `--help`, `man`, and `type`/`which`.
- Use the shell efficiently: history, tab-completion, editing, and clearing.
- Read the prompt and know "where" and "who" you are.

## Concepts
- **What the shell does:** it reads a line you type, splits it into words, finds the program named by the first word, runs it with the rest as arguments, shows its output, and waits for the next line. That's the whole loop.
- **Anatomy of a command:** `command [options] [arguments]`
  - **command** — the program (`ls`, `cat`, `grep`).
  - **options/flags** — change behavior. Short form `-l`, long form `--all`. Combine short flags: `ls -la` = `ls -l -a`. Some take a value: `--color=auto`, `-n 5`.
  - **arguments** — what to act on (file names, paths, text).
  - Example: `ls -l /etc` → program `ls`, option `-l` (long listing), argument `/etc` (the directory).
- **Reading the prompt:** a typical prompt is `user@host:cwd$`. e.g. `root@linux-lab:/work#`:
  - `root` = current user, `linux-lab` = hostname, `/work` = current directory.
  - The final character hints at privileges: **`#`** = root (superuser), **`$`** = normal user.
- **Getting help (do this constantly):**
  - `command --help` — quick usage summary (works for most tools).
  - `man command` — the full **manual page**. Scroll with arrows/space, search with `/word`, quit with `q`.
  - `type command` — is it a program, a shell builtin, or an alias?
  - `which command` — the full path of the program that would run.
  - `apropos keyword` — search man pages by keyword when you don't know the command's name.
- **Builtins vs programs.** Some commands (`cd`, `echo`, `pwd`, `export`) are **built into bash** itself; most (`ls`, `grep`, `cat`) are separate programs in `/usr/bin`. `type cd` vs `type ls` shows the difference. This matters because builtins have no man page of their own (`man bash`, or `help cd`, instead).
- **Efficiency features (these make you fast):**
  - **Tab completion** — start typing a command or path and press **Tab** to auto-complete; press it twice to list options. Use it always; it also prevents typos.
  - **History** — Up/Down arrows scroll previous commands. `history` lists them. `!!` re-runs the last command. `Ctrl+R` searches history interactively (type a few letters of an old command).
  - **Line editing** — `Ctrl+A` start of line, `Ctrl+E` end, `Ctrl+U` delete to start, `Ctrl+K` delete to end, `Ctrl+W` delete previous word.
  - **`Ctrl+C`** cancels the current command; **`Ctrl+L`** (or `clear`) clears the screen; **`Ctrl+D`** sends end-of-input (and at an empty prompt, logs out).
- **Whitespace & quoting (a preview):** the shell splits on spaces, so a filename with a space needs quotes: `cat "my notes.txt"`. We cover quoting properly in scripting (Step 19), but knowing this now avoids early frustration.

## Exercises
1. Run `whoami`, `hostname`, `pwd`, and `date`. Map each piece of your prompt to its meaning.
2. Run `ls`, then `ls -l`, then `ls -la`, then `ls -la /etc`. Describe what each flag/argument changed.
3. Get help three ways on `ls`: `ls --help`, `man ls` (scroll, search `/sort`, quit with `q`), and `type ls`. Then `type cd` and notice it's a builtin.
4. Use **Tab**: type `cat /etc/os` and press Tab to complete `os-release`. Then type `ls /et` + Tab.
5. Use **history**: run three commands, press Up to revisit them, then `Ctrl+R` and type `host` to find your `hostname` command. Run `!!` to repeat the very last command.
6. Use `apropos`: run `apropos directory` and skim what shows up — note that you can *discover* commands this way.
7. Deliberately make an error: run `lss` (typo). Read the "command not found" message and fix it. This is normal — get comfortable with it.

## Best Practices & Pitfalls
- **Lean on Tab completion and `Ctrl+R`.** Experienced users rarely type full paths or re-type old commands. Building these reflexes now pays off every single day.
- **Read error messages literally.** "No such file or directory", "permission denied", and "command not found" each point to a specific, different fix. Don't gloss over them.
- **`man` is your friend, not a last resort.** When a flag doesn't do what you expect, the answer is almost always in the man page. Searching with `/` is the trick that makes them usable.
- **Pitfall: spaces in names.** Unquoted spaces split into separate arguments. Quote paths with spaces, or avoid spaces in names you create.
- **Pitfall: assuming a command failed silently.** Many Unix tools print *nothing* on success (e.g. `cp`, `mv`). No news is good news. Step 08 covers exit codes for checking this precisely.

## Checklist
- [ ] I can break any command into program / options / arguments.
- [ ] I can read my prompt to know my user, host, and current directory.
- [ ] I can get help with `--help`, `man` (and quit with `q`), and `type`.
- [ ] Tab-completion, Up-arrow history, and `Ctrl+R` are becoming automatic.
- [ ] I understand the difference between a shell builtin and a program.

## Resources
- GNU Bash manual: https://www.gnu.org/software/bash/manual/bash.html
- "The Linux command line" (free book by William Shotts): https://linuxcommand.org/tlcl.php
- explainshell (paste a command, see each part explained): https://explainshell.com/
- man page reference: https://man7.org/linux/man-pages/
