# 09 — Text Processing

## Goals
- Search file contents fast with `grep`.
- Slice, transform, sort, and count text with `cut`, `sort`, `uniq`, `wc`, `tr`.
- Get a working introduction to `sed` (stream edit) and `awk` (column processing).
- Combine them into pipelines that answer real questions about logs and configs.

## Concepts
Linux is text-centric: configs, logs, and most data are plain text, so a small set of text tools handles an enormous amount of real work. These compose beautifully with pipes (Step 08).

- **`grep` — find lines matching a pattern** (your most-used text tool):
  - `grep "error" app.log` — lines containing "error".
  - `grep -i` — case-insensitive. `grep -r pattern dir/` — recursive search through a tree. `grep -n` — show line numbers. `grep -v` — *invert* (lines that DON'T match). `grep -c` — count matches. `grep -w` — whole word. `grep -E` — extended regex (alternation `a|b`, `+`, `?`).
  - `grep -A3 -B2 pattern` — show 3 lines After / 2 Before each match (context — invaluable in logs).
  - Combo you'll use daily: `grep -rn "TODO" /work/lab` or `tail -f error.log | grep -i timeout`.
- **`cut` — extract columns/fields:**
  - `cut -d: -f1 /etc/passwd` — split on `:`, take field 1 (usernames).
  - `cut -d, -f2,3 data.csv` — fields 2 and 3 of a CSV. `cut -c1-10` — characters 1–10.
- **`sort` — order lines:** `sort file`, `sort -r` (reverse), `sort -n` (numeric), `sort -k2` (by 2nd field), `sort -u` (sort + dedupe), `sort -t: -k3 -n /etc/passwd` (numeric sort by UID).
- **`uniq` — collapse adjacent duplicates** (input must be sorted first): `sort file | uniq`, `uniq -c` (count each), `sort file | uniq -c | sort -rn` (the classic "top N" idiom — count, then sort by count descending).
- **`wc` — count:** `-l` lines, `-w` words, `-c` bytes, `-m` chars.
- **`tr` — translate/delete characters:** `tr 'a-z' 'A-Z'` (uppercase), `tr -d '\r'` (strip carriage returns), `tr -s ' '` (squeeze repeated spaces).
- **`sed` — stream editor (find/replace on a stream):**
  - `sed 's/old/new/' file` — replace first `old` per line. `s/old/new/g` — all on each line. `sed -i` — edit the file **in place** (dangerous — back up first).
  - `sed -n '10,20p' file` — print only lines 10–20. `sed '/pattern/d'` — delete matching lines.
  - Web use: bump a version, comment out a config line, redact a token before sharing a log.
- **`awk` — field-aware processing** (a tiny language; you only need the basics):
  - `awk '{print $1}'` — first whitespace-separated field of each line. `$0` = whole line, `$NF` = last field.
  - `awk -F: '{print $1, $3}' /etc/passwd` — fields 1 and 3 with a `:` separator.
  - `awk '$3 > 1000' /etc/passwd` — lines where field 3 exceeds 1000. `awk '{sum+=$1} END {print sum}'` — sum a column.
  - Web use: extract status codes or response sizes from access logs and aggregate them.
- **The mental model:** `grep` filters *rows*, `cut`/`awk` select *columns*, `sort`/`uniq` aggregate, `wc` counts, `sed` rewrites. Pipe them in that order and you can interrogate almost any text data.

## Exercises
1. `grep`: `grep -i bash /etc/passwd`, then `grep -c "" /etc/passwd` (count lines), then `grep -v nologin /etc/passwd` (accounts that *can* log in). Try `grep -rn root /etc 2>/dev/null | head`.
2. `cut`: list all usernames with `cut -d: -f1 /etc/passwd`. Then their shells: `cut -d: -f1,7 /etc/passwd`.
3. Top-N idiom: `cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn` — which login shell is most common? Read each stage of the pipe.
4. `awk`: `awk -F: '{print $1 " has UID " $3}' /etc/passwd`. Then filter system vs human accounts: `awk -F: '$3 >= 1000 {print $1}' /etc/passwd`.
5. `sed`: copy a config first — `cp /etc/hostname /work/lab/h.txt` — then `sed 's/[a-z]/X/g' /work/lab/h.txt` (preview, no `-i`). Now try in-place on the copy: `sed -i 's/$/  # edited/' /work/lab/h.txt` and `cat` it. Never run `-i` on a real config without a backup.
6. `tr`: `echo "Hello World" | tr 'a-z' 'A-Z'`, and `cut -d: -f1 /etc/passwd | tr '\n' ' '` (usernames on one line).
7. Mini log analysis: create a fake access log and answer "top IPs":
   ```
   printf '10.0.0.1 GET /\n10.0.0.2 GET /a\n10.0.0.1 GET /b\n10.0.0.1 POST /c\n' > /work/lab/access.log
   awk '{print $1}' /work/lab/access.log | sort | uniq -c | sort -rn
   ```
   Then count POSTs: `grep -c POST /work/lab/access.log`. This is exactly how you'll read real nginx logs in Part 7.

## Best Practices & Pitfalls
- **`uniq` only dedupes *adjacent* lines — always `sort` first.** `sort file | uniq -c | sort -rn` is the idiom; `uniq` alone on unsorted input misses duplicates.
- **`sed -i` rewrites files in place with no undo.** Back up first (`cp file file.bak`), or test the expression without `-i` to preview, then add `-i`. On real configs, `diff` before/after.
- **`grep -r` can be slow/noisy on huge trees and may hit permission errors** — add `2>/dev/null` to suppress them, and consider `grep -rl` (just filenames) first.
- **Know when to reach for `awk` vs `cut`.** `cut` is simplest for fixed single-character delimiters; `awk` handles whitespace runs, conditions, and arithmetic. Don't write a 50-line `awk` program when a pipe of small tools is clearer.
- **Pitfall: regex special characters.** `.` `*` `[` `]` `$` mean something in patterns. `grep -F` (fixed strings) searches literally when you don't want regex; escape with `\` otherwise.
- **Pitfall: locale/encoding surprises** in `sort` and `tr`. For predictable byte-order sorting in scripts, prefix with `LC_ALL=C`.

## Checklist
- [ ] I can find lines with `grep` (`-i`, `-r`, `-n`, `-v`, `-c`, context `-A`/`-B`).
- [ ] I can extract columns with `cut -d -f` and `awk '{print $N}'`.
- [ ] I know the `sort | uniq -c | sort -rn` top-N idiom and why `sort` comes first.
- [ ] I can do a safe find/replace with `sed` (and why `-i` needs a backup).
- [ ] I can chain these to answer a real question about a log file.

## Resources
- `grep` manual: https://www.gnu.org/software/grep/manual/grep.html
- `sed` one-liners (classic reference): https://www.pement.org/sed/sed1line.txt
- `awk` in 20 minutes: https://ferd.ca/awk-in-20-minutes.html
- regexone (learn regex interactively): https://regexone.com/
