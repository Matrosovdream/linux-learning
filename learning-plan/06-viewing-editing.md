# 06 — Viewing & Editing Files

## Goals
- Read files several ways and pick the right tool for the size and purpose.
- Follow a growing log file in real time.
- Edit config files in the terminal with `nano`, and survive `vim` when it's the only editor available.

## Concepts
- **Viewing whole files:**
  - `cat file` — dump the entire file to the screen. Great for short files; overwhelming for big ones. `cat -n` adds line numbers.
  - `less file` — a **pager**: scroll a big file without loading it all. Arrows/space to move, `/term` to search, `n`/`N` for next/prev match, `g`/`G` for top/bottom, `q` to quit. This is the right tool for logs and long configs. (`more` is the older, weaker version.)
- **Viewing parts:**
  - `head file` — first 10 lines. `head -n 20` for 20. Good for "what's at the top of this config."
  - `tail file` — last 10 lines. `tail -n 50`. Logs put the newest entries at the bottom, so `tail` is how you see recent activity.
  - **`tail -f file`** — *follow*: keep printing new lines as they're appended. This is *the* way to watch a log live while you hit a web server. `Ctrl+C` to stop. `tail -F` also survives log rotation.
- **Quick whole-file info:** `wc -l file` (line count), `wc -c` (bytes). `file name` (what kind of file).
- **Editing — `nano` (beginner-friendly):**
  - `nano file` opens an editor with on-screen shortcut hints. `^` means **Ctrl**.
  - Save: `Ctrl+O` then Enter. Exit: `Ctrl+X`. Cut line: `Ctrl+K`. Paste: `Ctrl+U`. Search: `Ctrl+W`.
  - Recommended default editor while learning — you can see every command on screen.
- **Editing — `vim` (everywhere, worth surviving):**
  - `vim` is **modal**: you start in **Normal** mode (keys are commands, not text). Press `i` to enter **Insert** mode and type normally. Press `Esc` to return to Normal mode.
  - Save & quit: from Normal mode type `:wq` then Enter. Quit without saving: `:q!` then Enter. (Memorize these two — they get you out of any vim you opened by accident.)
  - You don't need vim mastery now; you need to *not get stuck*. Many minimal servers/containers ship only `vi`/`vim`, so knowing `Esc :wq` / `Esc :q!` is essential.
- **Choosing your editor:** set `EDITOR` so tools (like `crontab -e` later) open your preferred one: `export EDITOR=nano`. We make this permanent in Step 18.
- **Comparing files:** `diff a b` shows line-by-line differences — handy when you back up a config (`cp nginx.conf nginx.conf.bak`), edit it, then `diff` to see exactly what you changed.

## Exercises
1. `cat /etc/os-release` (short — fine for `cat`). Then `cat -n /etc/services` (long — notice why `cat` is the wrong tool here) and `Ctrl+C` if needed.
2. Page a long file: `less /etc/services`. Practice: space to scroll, `/tcp` to search, `n` for next, `G` to jump to end, `g` to top, `q` to quit.
3. `head -n 5 /etc/passwd` and `tail -n 5 /etc/passwd`. Then `wc -l /etc/passwd` — how many user accounts exist on a fresh box?
4. Create and edit with nano: `nano /work/lab/notes.md`, type a few lines, save with `Ctrl+O`/Enter, exit with `Ctrl+X`. Reopen and confirm it saved.
5. Survive vim: `vim /work/lab/vimtest.txt`, press `i`, type a line, press `Esc`, then `:wq` Enter. Reopen with `vim`, then exit *without* changes via `:q!`.
6. Backup-edit-diff: `cp /etc/hostname /work/lab/hostname.orig`, edit `/work/lab/hostname.orig` in nano, then `diff /etc/hostname /work/lab/hostname.orig` to see your change.
7. Live log preview (rehearsal for later): run `tail -f /work/lab/notes.md` in your shell; in a *second* shell into the container, append with `echo "new line" >> /work/lab/notes.md` and watch it appear. `Ctrl+C` to stop following.

## Best Practices & Pitfalls
- **Use `less` for anything that might be long.** `cat`-ing a huge log floods your terminal and scrolls past the part you need. `less` (or `tail`) is almost always the right call for logs and big configs.
- **Always back up a config before editing it:** `cp file file.bak`. If your edit breaks a service, you can restore in one `cp` — and `diff` tells you what changed.
- **`tail -f` is your live debugging window.** Keep it open on `/var/log/nginx/error.log` while testing requests and you'll see problems the instant they happen (Steps 22–23).
- **Pitfall: getting trapped in vim.** If you opened vim by accident: `Esc` then `:q!` then Enter. That's the universal escape hatch.
- **Pitfall: editing a file you don't have permission to save.** nano/vim will let you edit but fail to write. You'll need `sudo nano file` (covered in Step 07/12). The editor usually warns you the file is read-only.
- **Pitfall: line endings.** Files created on Windows can carry `\r\n`; on Linux that breaks scripts/configs. `file name` and `cat -A` reveal it; `dos2unix` fixes it.

## Checklist
- [ ] I use `cat` for short files and `less`/`tail` for long ones.
- [ ] I can `tail -f` a file and watch new lines arrive live.
- [ ] I can create, edit, and save a file in `nano`.
- [ ] I can enter insert mode, save, and quit `vim` (`i`, `Esc`, `:wq`, `:q!`) without getting stuck.
- [ ] I back up configs before editing and use `diff` to confirm changes.

## Resources
- nano cheat sheet: https://www.nano-editor.org/dist/latest/cheatsheet.html
- Interactive vim tutorial: run `vimtutor` in your lab (about 30 min, highly recommended)
- `man less` (the pager you'll use most)
- `diff` manual: https://man7.org/linux/man-pages/man1/diff.1.html
