# 08 — Redirection, Pipes & Streams

## Goals
- Understand the three standard streams: stdin, stdout, stderr.
- Redirect output to files and read input from files.
- Chain commands with pipes to build powerful one-liners.
- Check whether a command succeeded using its exit code.

## Concepts
- **Three standard streams.** Every program is born with three connections:
  - **stdin (0)** — standard input (where it reads input; default: your keyboard).
  - **stdout (1)** — standard output (normal results; default: your screen).
  - **stderr (2)** — standard error (error/diagnostic messages; default: your screen too).
  - Keeping stdout and stderr separate is deliberate: you can capture results while still seeing errors (or vice versa).
- **Redirecting output to files:**
  - `>` — redirect stdout to a file, **overwriting** it: `ls > files.txt`.
  - `>>` — redirect stdout, **appending**: `echo "line" >> log.txt`.
  - `2>` — redirect **stderr**: `command 2> errors.txt`.
  - `2>&1` — send stderr to wherever stdout currently goes: `command > all.txt 2>&1` (capture both).
  - `&>` — shorthand for "both stdout and stderr": `command &> all.txt`.
  - `/dev/null` — the black hole. `command 2>/dev/null` discards errors; `command >/dev/null 2>&1` discards everything (run for side effects only).
- **Redirecting input:**
  - `<` — feed a file as stdin: `sort < names.txt`.
  - **Here-doc** `<<EOF ... EOF` — feed a multi-line block as stdin (great for writing config files in scripts).
  - **Here-string** `<<<"text"` — feed a single string as stdin.
- **Pipes `|` — the heart of Unix.** Connect one command's stdout to the next command's stdin: `command1 | command2`. This lets you compose small tools into big results without temp files:
  - `ls -l | grep ".log"` — list files, keep only lines mentioning `.log`.
  - `cat access.log | grep 404 | wc -l` — count 404s in a log.
  - `ps aux | grep nginx` — find nginx processes.
  - `history | tail -20` — last 20 commands.
- **`tee` — split the stream.** `command | tee file` sends output to *both* the screen and a file (append with `tee -a`). Useful to watch output live *and* save it: `make 2>&1 | tee build.log`.
- **Exit codes (`$?`).** Every command returns a number when it finishes: **`0` = success**, non-zero = failure (the specific number is the kind of failure). Check the last command with `echo $?`.
  - Chain on success: `cmd1 && cmd2` runs `cmd2` only if `cmd1` succeeded.
  - Chain on failure: `cmd1 || cmd2` runs `cmd2` only if `cmd1` failed.
  - Sequence regardless: `cmd1 ; cmd2` runs both no matter what.
  - This is the basis of safe scripting (Step 19) and is why `mkdir build && cd build` is a common idiom.
- **Why this matters for web dev:** reading and filtering logs (`tail -f error.log | grep -i timeout`), capturing command output for debugging, and writing config files from scripts all rely on streams and redirection. It's the glue of server work.

## Exercises
1. Stdout to file: `ls -la /etc > /work/lab/etc-listing.txt`, then `less /work/lab/etc-listing.txt`. Re-run with `>` and note it overwrote; run with `>>` and note it appended.
2. Separate the streams: run `ls /etc /nonexistent` and watch both a listing (stdout) and an error (stderr) appear. Now `ls /etc /nonexistent > out.txt 2> err.txt` and inspect each file — the listing went to `out.txt`, the error to `err.txt`.
3. Discard noise: `ls /etc /nonexistent 2>/dev/null` (errors gone, listing remains). Then `... >/dev/null 2>&1` (silent).
4. Build a pipeline: `cat /etc/passwd | grep bash | wc -l` — how many accounts use bash as their shell? Break it into steps to see each stage's output.
5. `tee` it: `ls -l /usr/bin | tee /work/lab/bin-list.txt | wc -l` — see the count on screen *and* save the full list to a file.
6. Here-doc: write a small file in one shot:
   ```
   cat > /work/lab/site.conf <<EOF
   server_name example.local;
   root /var/www/html;
   EOF
   ```
   Then `cat /work/lab/site.conf`. (You'll use this pattern to generate configs.)
7. Exit codes: run `ls /etc; echo $?` (expect 0), then `ls /nope; echo $?` (expect non-zero). Try `mkdir /work/lab/built && cd /work/lab/built && pwd` and explain why `&&` is safer than `;` here.

## Best Practices & Pitfalls
- **`>` overwrites without warning.** `command > important.txt` destroys the old contents instantly. Use `>>` to append, and double-check the filename. (`set -o noclobber` can block accidental overwrites.)
- **Order matters in `> file 2>&1`.** `2>&1 > file` does *not* do what you want — it points stderr at the *old* stdout (the screen) before stdout is redirected. Put `> file` first, then `2>&1`.
- **Prefer pipes over temp files.** `cmd1 | cmd2` is cleaner, faster, and leaves no mess compared to `cmd1 > tmp; cmd2 < tmp; rm tmp`.
- **`cat file | grep x` works but `grep x file` is better.** A "useless use of cat" is harmless while learning, but most tools take a filename directly. Pipe when you genuinely need to chain.
- **Pitfall: thinking no output means failure.** Many commands are silent on success. Check `$?` (0 = fine) rather than assuming.
- **Pitfall: redirecting into the file you're reading.** `sort file > file` truncates the file before `sort` reads it — you lose the data. Write to a new file, then `mv` it.

## Checklist
- [ ] I can explain stdin, stdout, stderr and their numbers (0, 1, 2).
- [ ] I can redirect stdout (`>`, `>>`), stderr (`2>`), and both (`> f 2>&1`).
- [ ] I can build multi-stage pipelines with `|` and use `tee` to save mid-stream.
- [ ] I know `/dev/null` discards output and a here-doc (`<<EOF`) feeds multi-line input.
- [ ] I can read `$?` and use `&&`, `||`, `;` to chain commands by success/failure.

## Resources
- Bash redirections (manual): https://www.gnu.org/software/bash/manual/bash.html#Redirections
- `man bash` → "REDIRECTION" section
- Pipes & filters (Software Carpentry): https://swcarpentry.github.io/shell-novice/04-pipefilter.html
- Exit status explained: https://tldp.org/LDP/abs/html/exit-status.html
