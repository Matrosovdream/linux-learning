# 11 — Processes, Jobs & Signals

## Goals
- Understand what a process is and inspect what's running.
- Control processes: foreground/background jobs, and stopping them with signals.
- Read resource usage (CPU, memory) to spot a runaway service.

## Concepts
- **What's a process?** A running program. Each has a **PID** (process ID), a **parent** (PPID), an owner (user), and its own memory. The first process is `init`/`systemd` (PID 1); everything else descends from it. In a container, PID 1 is whatever the container was told to run.
- **Listing processes:**
  - `ps aux` — snapshot of **all** processes (BSD style): columns USER, PID, %CPU, %MEM, VSZ/RSS (memory), STAT (state), START, TIME, COMMAND. The everyday "what's running" command.
  - `ps -ef` — same idea, System-V style (shows PPID). Pick whichever you like.
  - `ps aux | grep nginx` — find a specific process (you'll do this constantly). `pgrep nginx` returns just PIDs.
  - `pstree` — processes as a parent/child tree (shows what spawned what).
- **Live monitoring:**
  - `top` — live, updating table sorted by CPU. Keys: `q` quit, `M` sort by memory, `P` by CPU, `k` kill a PID. Read load average and per-process %CPU/%MEM here.
  - `htop` — nicer `top` with colors, scrolling, and mouse (`apt install htop`). Preferred when available.
- **Process states (STAT column):** `R` running, `S` sleeping (waiting), `D` uninterruptible sleep (usually I/O), `Z` zombie (finished but not reaped by parent), `T` stopped. A pile of zombies or `D` usually signals a problem.
- **Jobs: foreground vs background** (within one shell):
  - A command runs in the **foreground** and ties up your prompt. Append `&` to run in the **background**: `sleep 300 &`.
  - `Ctrl+Z` suspends the foreground job; `bg` resumes it in the background; `fg` brings it back to the foreground.
  - `jobs` lists this shell's jobs; refer to them as `%1`, `%2`.
  - **`nohup cmd &`** and `disown` let a process keep running after you log out (closing the shell normally sends it a hangup). For real services you'll use systemd instead (Step 14), but `nohup` is handy for quick long-running tasks.
- **Signals — how you talk to a process:**
  - `kill <PID>` sends **SIGTERM (15)** by default: "please shut down cleanly." Most well-behaved services flush and exit.
  - `kill -9 <PID>` sends **SIGKILL (9)**: forced, immediate, uncatchable. Last resort — the process can't clean up (may leave temp files, corrupt state). Try SIGTERM first.
  - `kill -HUP <PID>` (**SIGHUP, 1**) often means "reload your config" for daemons like nginx — no downtime.
  - `kill -STOP`/`-CONT` pause/resume. `Ctrl+C` in the foreground sends **SIGINT (2)**.
  - `kill -l` lists all signals. `pkill nginx` / `killall nginx` kill by name instead of PID.
- **Resource basics:** `free -h` (memory), `uptime` (load average — roughly how many processes are waiting to run), `nproc` (CPU count), `df -h` (disk space), `du -sh dir` (a directory's size). On real boxes, "site is slow" investigations start here.
- **Web relevance:** find a stuck app process, reload nginx without dropping connections (`kill -HUP` / `nginx -s reload`), spot the service eating all the RAM, confirm your app is actually listening (cross-reference with ports in Step 16).

## Exercises
1. Snapshot: `ps aux | head`, then `ps aux | grep bash`. Identify your own shell's PID. Run `echo $$` — that prints the current shell's PID; find it in the `ps` output.
2. Tree: `apt install -y psmisc` (for `pstree`) then `pstree -p`. See how processes descend from PID 1.
3. Background a job: `sleep 600 &`, note the PID, run `jobs`, then `ps aux | grep sleep`. Bring it forward with `fg`, suspend with `Ctrl+Z`, resume in background with `bg`.
4. Signals: start `sleep 600 &`, then `kill <PID>` (SIGTERM) and confirm it's gone with `jobs`/`ps`. Start another and try `kill -9 <PID>`. Discuss with Claude when `-9` is justified.
5. `pkill`/`pgrep`: start two `sleep 600 &` jobs, then `pgrep sleep` (lists PIDs) and `pkill sleep` (kills all). Verify.
6. Live view: install and run `htop` (`apt install -y htop`). Sort by memory (`F6`/`M`), find the biggest process, quit with `q`. Then `free -h`, `uptime`, `df -h`, `nproc`.
7. Make a "runaway" and catch it: run `yes > /dev/null &` (pins a CPU), watch it in `htop`/`top` at ~100% CPU, then kill it. (`yes` floods stdout; `/dev/null` swallows it.) This rehearses diagnosing a pegged service.

## Best Practices & Pitfalls
- **Always try `kill` (SIGTERM) before `kill -9` (SIGKILL).** SIGTERM lets a service flush buffers, finish requests, and remove its PID/lock file. SIGKILL can leave corruption or stale locks. `-9` is for processes that ignore everything else.
- **Find the *right* PID before killing.** `pkill`/`killall` match by name and can hit more than you intend (e.g. `pkill python` kills *every* python). Prefer a specific PID, or use `pgrep -a name` to see exactly what you'd hit.
- **For services, reload beats restart.** `kill -HUP` (or `systemctl reload`) re-reads config without dropping connections; a full restart causes a brief outage.
- **Zombies aren't killable** — they're already dead, waiting on their parent to reap them. Fix the parent (or it's harmless and clears when the parent exits). Don't `kill -9` a zombie expecting it to vanish.
- **Pitfall: background jobs die when the shell closes.** A plain `cmd &` gets SIGHUP on logout. Use `nohup`, `disown`, or (properly) a systemd service for anything that must survive.
- **Pitfall: reading memory columns wrong.** RSS is real resident memory; VSZ is virtual address space and is often huge and misleading. Watch RSS / `%MEM`.

## Checklist
- [ ] I can list and find processes with `ps aux | grep`, `pgrep`, and `pstree`.
- [ ] I can monitor live load with `top`/`htop` and read `%CPU`, `%MEM`, load average.
- [ ] I can background (`&`), suspend (`Ctrl+Z`), and resume (`bg`/`fg`) jobs.
- [ ] I can stop a process with SIGTERM and know when SIGKILL (`-9`) is warranted.
- [ ] I know `kill -HUP`/reload re-reads config without downtime.

## Resources
- `man ps`, `man kill`, `man top`
- Signals reference: https://man7.org/linux/man-pages/man7/signal.7.html
- htop guide: https://htop.dev/
- Understanding load average: https://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html
