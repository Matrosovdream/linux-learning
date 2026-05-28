# 10 — Finding Things

## Goals
- Locate files by name, type, size, time, and permission with `find`.
- Act on the files you find (delete, copy, chmod) safely.
- Understand shell globbing vs regex, and have a working grasp of basic regex.

## Concepts
- **`find` — the workhorse for locating files.** Syntax: `find <where> <tests> <action>`.
  - `find /etc -name "*.conf"` — by name (quote the glob so the shell doesn't expand it first). `-iname` is case-insensitive.
  - `find . -type f` (files), `-type d` (directories), `-type l` (symlinks).
  - `find /var/log -size +10M` — larger than 10 MB (`-size -1k`, `+100M`, etc.).
  - `find . -mtime -1` — modified in the last day (`-mtime +7` = older than 7 days; `-mmin -30` = last 30 min). Great for "what changed recently?"
  - `find . -perm 600`, `find . -user www-data`, `find . -group www-data` — by permission/owner.
  - Combine tests (implicit AND): `find /var/www -type f -name "*.php" -mtime -2`.
  - `-maxdepth N` limits how deep it recurses (put it *before* other tests).
- **Acting on results:**
  - `-delete` — delete matches: `find . -name "*.tmp" -delete` (preview with `-print` first!).
  - `-exec cmd {} \;` — run a command per match (`{}` = the file): `find . -name "*.sh" -exec chmod +x {} \;`.
  - `-exec cmd {} +` — batch many files into one invocation (faster): `find . -name "*.log" -exec rm {} +`.
  - Pipe to `xargs`: `find . -name "*.log" | xargs rm` (use `find -print0 | xargs -0` to handle spaces in names safely).
- **`locate` — instant name search from a prebuilt index.** `locate nginx.conf`. Much faster than `find` for "where is this file by name," but relies on a database refreshed by `updatedb` (run it once; may need `apt install plocate`). Use `find` for fresh/precise queries, `locate` for quick lookups.
- **`which` / `type` / `command -v`** — find an *executable* in your `PATH`: `which nginx`. (Step 18 covers `PATH`.)
- **Globbing (shell) vs regex (tools) — don't confuse them:**
  - **Globs** are expanded by the *shell* against filenames: `*` = any chars, `?` = one char, `[a-z]` = a range, `{a,b}` = brace expansion. Used in `ls *.log`, `rm tmp_*`.
  - **Regex** is matched by *tools* (`grep`, `sed`, `awk`) against text content, and is more powerful/different in meaning.
- **Basic regex (for `grep`/`sed`):**
  - `.` any single char · `*` zero-or-more of the previous · `^` start of line · `$` end of line.
  - `[abc]` one of a set · `[^abc]` none of a set · `[0-9]` a digit range.
  - Extended (`grep -E` / `sed -E`): `+` one-or-more · `?` optional · `|` alternation · `()` grouping · `{n,m}` repetition.
  - Examples: `grep "^root" /etc/passwd` (lines starting with root), `grep -E "error|fail|warn" app.log`, `grep -E "[0-9]{3}\.[0-9]+" file` (IP-ish).
- **Web use:** "find every config that mentions port 8080," "delete logs older than 14 days," "which uploaded files are world-writable," "where did apt install the nginx binary." All are one-liners with `find` + `grep`.

## Exercises
1. By name: `find /etc -name "*.conf" 2>/dev/null | head`. Then case-insensitive: `find /etc -iname "*HOSTNAME*" 2>/dev/null`.
2. By type & depth: `find / -maxdepth 2 -type d 2>/dev/null | head -30` — top-level structure as a list. Then `find /work -type f`.
3. By time: create a file, then `find /work -mmin -5` (things changed in the last 5 minutes). Make an old-looking file with `touch -d "10 days ago" /work/lab/old.txt` and try `find /work -mtime +7`.
4. By size: `find /usr -type f -size +5M 2>/dev/null | head` — the biggest installed files.
5. Safe `-exec`: create some scripts, then `find /work -name "*.sh" -exec chmod +x {} \;` and verify with `ls -l`. Always run with `-print` first to preview matches.
6. Safe delete: `touch /work/lab/{1,2,3}.tmp`, preview with `find /work -name "*.tmp" -print`, then `find /work -name "*.tmp" -delete` and confirm they're gone.
7. `locate` vs `find`: `apt install -y plocate && updatedb`, then `locate os-release` (instant) vs `find / -name os-release 2>/dev/null` (slower, live). Ask Claude when each is the right choice.
8. Glob vs regex: `ls /etc/*.conf` (shell glob) vs `ls /etc | grep -E "\.conf$"` (regex on text). Both list `.conf` files — note they're doing different things.

## Best Practices & Pitfalls
- **Always preview before `find ... -delete` or `-exec rm`.** Run the same `find` with `-print` first and read the list. `find` will happily delete thousands of files instantly.
- **Quote name patterns:** `find . -name "*.log"` — without quotes the shell expands `*.log` against the *current* directory before `find` runs, giving wrong results.
- **Use `-print0 | xargs -0` (or `-exec ... +`) when names may contain spaces/newlines.** Plain `xargs` splits on whitespace and mangles such names.
- **Don't `find /` without `2>/dev/null` and a `-maxdepth`/filter** unless you mean it — it walks the entire tree (including slow/virtual paths) and floods you with permission errors.
- **Pitfall: confusing globs and regex.** `*` means "zero-or-more of the previous char" in regex but "any string" as a glob. `grep *.log` is almost never what you want — that's a glob in a regex slot.
- **Pitfall: `locate` showing stale results.** Its index is only as fresh as the last `updatedb`. For files created seconds ago, use `find`.

## Checklist
- [ ] I can find files by name, type, size, mtime, and owner/permission with `find`.
- [ ] I can act on matches with `-delete` and `-exec`, and I preview with `-print` first.
- [ ] I can explain the difference between a shell glob and a regex.
- [ ] I can read/write basic regex (`^ $ . * [] | +`) for `grep`/`sed`.
- [ ] I know when to use `locate` vs `find` vs `which`.

## Resources
- `man find` (read the EXAMPLES section — it's excellent)
- `find` tutorial: https://www.redhat.com/sysadmin/linux-find-command
- regex cheat sheet: https://quickref.me/regex
- `xargs` explained: https://man7.org/linux/man-pages/man1/xargs.1.html
