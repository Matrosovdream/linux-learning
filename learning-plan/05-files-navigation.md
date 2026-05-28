# 05 — Navigating & Managing Files

## Goals
- Move around the filesystem confidently with `cd`, `pwd`, `ls`.
- Create, copy, move, rename, and delete files and directories — safely.
- Understand links (`ln`), wildcards, and how to inspect what you're about to touch.

## Concepts
- **Where am I / what's here:**
  - `pwd` — print working directory (your current absolute path).
  - `ls` — list. Key flags: `-l` (long: permissions, owner, size, date), `-a` (include hidden dotfiles), `-h` (human-readable sizes, pairs with `-l`), `-t` (sort by time), `-R` (recursive), `-F` (mark dirs with `/`). Common combo: `ls -lah`.
  - `tree` — show a directory as a tree (may need `apt install tree`). `tree -L 2` limits depth.
- **Moving around:**
  - `cd path` — change directory. `cd` alone → home. `cd -` → previous directory. `cd ..` → up one. `cd /etc/nginx` → absolute jump.
  - Tab-complete paths as you type — faster and typo-proof.
- **Creating:**
  - `mkdir name` — make a directory. `mkdir -p a/b/c` — create nested parents in one go (you'll use `-p` constantly).
  - `touch file` — create an empty file (or update its timestamp if it exists).
- **Copying & moving:**
  - `cp src dst` — copy a file. `cp -r srcdir dstdir` — copy a directory recursively. `cp -i` — prompt before overwriting (safe). `cp -a` — archive copy that preserves permissions/timestamps (good for deploys).
  - `mv src dst` — move **or rename** (same command for both). `mv old.txt new.txt` renames; `mv file.txt dir/` moves. `mv -i` prompts before clobbering.
- **Deleting (the dangerous part):**
  - `rm file` — remove a file. `rm -r dir` — remove a directory and its contents. `rm -f` — force (no prompts, ignore missing). `rm -i` — prompt for each.
  - **There is no trash/undo.** `rm` is permanent. Treat `rm -rf` with respect (see pitfalls).
  - `rmdir dir` — removes an *empty* directory only (a safer way to delete dirs you think are empty).
- **Wildcards (globbing)** — the shell expands these before the command runs:
  - `*` = any characters (`*.log` = all files ending in `.log`), `?` = one character, `[abc]` = one of a set, `{a,b}` = brace expansion (`file.{txt,bak}`).
  - Test a glob safely with `ls` or `echo` *before* using it with `rm`: `echo *.log` shows exactly what would match.
- **Links:**
  - **Symbolic link (symlink):** `ln -s /target /link` — a pointer to another path (like a shortcut). Used everywhere in Linux (e.g. enabling an nginx site by symlinking a config). `ls -l` shows `link -> target`.
  - **Hard link:** `ln target link` — a second name for the same data; rarer in day-to-day work.
- **Inspecting before acting:** `file name` tells you what a file actually is (text, binary, image). `stat name` shows size, timestamps, permissions, inode. `du -sh dir` shows a directory's total size.

## Exercises
1. Navigate: from `/`, `cd /etc`, run `pwd`, `cd ..`, `pwd`, then `cd -` to bounce back. Then `cd ~` to go home and `pwd`.
2. Build a workspace in your persistent area: `mkdir -p /work/lab/{configs,logs,sites}` then `ls -R /work/lab`. (Notice brace expansion made three subdirs at once.)
3. Create and inspect: `touch /work/lab/sites/index.html`, then `ls -lah /work/lab/sites`, `file /work/lab/sites/index.html`, `stat /work/lab/sites/index.html`.
4. Copy & rename: `cp /etc/hostname /work/lab/configs/`, then `mv /work/lab/configs/hostname /work/lab/configs/hostname.bak`. Verify with `ls -l`.
5. Glob safely: create `touch /work/lab/logs/{a,b,c}.log`, then `echo /work/lab/logs/*.log` to preview the match, *then* `ls /work/lab/logs/*.log`.
6. Symlink: `ln -s /work/lab/sites /work/lab/current-site`, then `ls -l /work/lab` and confirm the `->` arrow. Ask Claude how nginx uses this pattern (`sites-available` → `sites-enabled`).
7. Delete carefully: remove just the `.log` files with `rm /work/lab/logs/*.log`, then remove the now-empty dir with `rmdir /work/lab/logs`. Confirm with `ls`.

## Best Practices & Pitfalls
- **`rm -rf` is the most dangerous command you'll learn.** It deletes recursively and silently. Before running it, `ls` the same path first, and never run it with a variable you haven't checked (`rm -rf "$DIR/"` when `$DIR` is empty becomes `rm -rf /`). In the lab the blast radius is just the container — practice caution here so it's a habit on real servers.
- **Preview globs with `echo` or `ls`** before pairing them with `rm` or `mv`. The shell expands `*` — make sure it expands to what you think.
- **Use `-i` while learning.** `cp -i`, `mv -i`, `rm -i` prompt before destroying data. You can drop them once you're confident.
- **`mv` is rename *and* move.** There's no separate `rename` you need for the basics.
- **Pitfall: `cp` a directory without `-r`.** You'll get "omitting directory". Add `-r` (or `-a` to preserve metadata).
- **Pitfall: trailing slashes with symlinks and `mv`.** `mv file dir` vs `mv file dir/` differ when `dir` doesn't exist — the slash asserts "this is a directory."

## Checklist
- [ ] I can navigate with `cd` (including `cd -`, `cd ..`, `cd ~`) and always know where I am with `pwd`.
- [ ] I can create nested directories with `mkdir -p` and files with `touch`.
- [ ] I can copy (`cp -r`), move/rename (`mv`), and delete (`rm`, `rmdir`) confidently.
- [ ] I preview globs with `echo`/`ls` before destructive use.
- [ ] I can create and recognize a symlink, and explain one real web use for it.

## Resources
- `man ls`, `man cp`, `man mv`, `man rm` (read the OPTIONS sections)
- Linux command line basics (Shotts, chs. 2–4): https://linuxcommand.org/tlcl.php
- Glob patterns explained: https://www.gnu.org/software/bash/manual/bash.html#Filename-Expansion
- Symbolic vs hard links: https://man7.org/linux/man-pages/man1/ln.1.html
