# 20 — Scheduling & Logs

## Goals
- Schedule recurring tasks with `cron` (and know `systemd timers` exist).
- Find, read, and follow system and application logs.
- Keep a server healthy: rotate logs, watch disk usage, avoid the "disk full from logs" outage.

## Concepts
- **`cron` — run commands on a schedule.** A background daemon (`cron`) reads **crontab** files and runs jobs at the specified times. Per-user crontabs are edited with `crontab -e` (opens your `$EDITOR`), listed with `crontab -l`, removed with `crontab -r`.
  - **Crontab line format:** `minute hour day-of-month month day-of-week command`
    ```
    # ┌── min (0-59)
    # │ ┌── hour (0-23)
    # │ │ ┌── day of month (1-31)
    # │ │ │ ┌── month (1-12)
    # │ │ │ │ ┌── day of week (0-7, 0/7=Sun)
    # * * * * * command
    ```
  - `*` = every. `*/5 * * * *` = every 5 minutes. `0 3 * * *` = 3:00 AM daily. `0 0 * * 0` = midnight Sunday. `@daily`, `@hourly`, `@reboot` are shortcuts.
  - System-wide crontabs: `/etc/crontab` and drop-in dirs `/etc/cron.d/`, plus `/etc/cron.{daily,weekly,monthly}/` (put a script there and it runs on that cadence).
  - Use https://crontab.guru/ to read/write schedules until the syntax sticks.
- **The cron environment trap (read this twice).** cron runs jobs with a **minimal environment**: a bare `PATH` (often just `/usr/bin:/bin`), no `.bashrc`, and `HOME`/`SHELL` defaults. Scripts that work in your terminal fail under cron because a command isn't found or a variable is unset. Fixes: use **absolute paths** for every command/file, set needed env at the top of the script (or in the crontab), and **redirect output to a log** so you can see failures: `*/5 * * * * /usr/local/bin/job.sh >> /var/log/job.log 2>&1`.
- **`systemd timers`** are the modern alternative to cron (a `.timer` + `.service` pair). They integrate with `journalctl`, handle missed runs (`Persistent=true`), and are easier to debug — but require systemd (Step 14's container caveat applies). Know they exist; cron is universal and fine to start with.
- **Where logs live & how to read them:**
  - **`/var/log/`** — the traditional home: `syslog`/`messages` (general system), `auth.log` (logins, sudo, SSH — security-relevant), `nginx/access.log` & `nginx/error.log`, plus per-service dirs. Read with `less`, `tail -n 100`, and **`tail -f`** to watch live (Step 06).
  - **The systemd journal** — on systemd hosts, services log here; read with `journalctl -u <svc>` (Step 14). Many modern setups use both.
  - **Filtering logs** ties back to Step 09: `grep -i error /var/log/syslog`, `awk '{print $9}' access.log | sort | uniq -c | sort -rn` (top status codes), `tail -f error.log | grep -i timeout`.
  - **Containers log to stdout/stderr by convention**, captured by the runtime: `docker compose logs -f <service>`. That's the container equivalent of `tail -f /var/log/...` — your app shouldn't write to log files inside a container, it should print to stdout.
- **Log rotation — why servers don't fill their disks.** Unbounded logs eventually consume all disk and crash services. **`logrotate`** (run daily via cron) rotates logs: renames `access.log` → `access.log.1`, compresses old ones (`.gz`), deletes ones past a retention limit. Config in `/etc/logrotate.conf` and `/etc/logrotate.d/<service>`. Packages like nginx ship their own rotation rules there.
- **Keeping a server tidy (the ops mindset):** watch disk with `df -h` (and `du -sh /var/log/*` to find the hog), confirm scheduled jobs actually ran (check their log + exit status), and ensure logs rotate. "Disk full because of logs" is one of the most common real-world outages — and entirely preventable.

## Exercises
1. Set up cron in the lab: `apt install -y cron && service cron start` (containers may not start it automatically). Confirm with `ss`/`ps` that `cron` is running.
2. Your first job: `crontab -e`, add `* * * * * date >> /work/cron-test.log 2>&1`, save. Wait a couple minutes, then `cat /work/cron-test.log` — watch entries accumulate. `crontab -l` to view, then remove the line.
3. Read a schedule: on paper, translate `30 2 * * 1-5`, `*/15 * * * *`, and `0 0 1 * *` into English. Check yourself on crontab.guru.
4. Reproduce the cron-environment trap: write a script that relies on a command via a custom `PATH` (e.g. `~/bin/hello` from Step 18) with **no** absolute path, schedule it, and watch it fail in the log. Fix it by using the absolute path and/or setting `PATH` at the top. Internalize the lesson.
5. Explore logs: `ls -lah /var/log`, `tail -n 30 /var/log/auth.log` (look for your `sudo`/`su` activity from earlier lessons), `grep -i error /var/log/*log 2>/dev/null | head`.
6. Live log analysis with nginx (if installed from earlier): hit it a few times with `curl http://127.0.0.1` (and some 404s like `curl http://127.0.0.1/nope`), then `tail -n 20 /var/log/nginx/access.log` and run `awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn` to tally status codes.
7. Inspect rotation: `cat /etc/logrotate.conf`, `ls /etc/logrotate.d/`, and read the nginx rule if present. Discuss with Claude what `rotate 14`, `daily`, and `compress` mean and how this prevents a disk-full outage.

## Best Practices & Pitfalls
- **Always redirect cron job output to a log (`>> file 2>&1`).** By default cron emails output (which goes nowhere on most boxes), so failures vanish silently. A log file is how you find out a 3 AM backup has been broken for a month.
- **Use absolute paths in cron jobs and scripts.** cron's bare `PATH` won't find `nginx`, `docker`, or your `~/bin` tools. Write `/usr/sbin/nginx`, set `PATH` at the top of the script, and don't assume any interactive setup.
- **Make sure logs rotate before they fill the disk.** A runaway `error.log` filling `/` takes the whole server down. Confirm `logrotate` covers your app's logs; for custom logs, add a `/etc/logrotate.d/` rule.
- **Check `auth.log` for security events.** Failed SSH logins, unexpected `sudo` use, and new sessions show up here — it's the first place to look if a box seems compromised.
- **Pitfall: testing a cron job only by running it in your shell.** It works for you (full environment) but fails under cron (minimal environment). Test by *actually scheduling it* and reading its log, or run it with a stripped env: `env -i /bin/sh -c '/path/job.sh'`.
- **Pitfall: app writing logs to files inside a container.** Containers are ephemeral and you want centralized logs — print to stdout/stderr and let `docker compose logs` / your log driver handle it. Don't reinvent `/var/log` inside an image.
- **Pitfall: cron daemon not running in a container.** Unlike a VM, a container won't start `cron` for you. If a scheduled job "never runs," check the daemon is actually up.

## Checklist
- [ ] I can read and write a crontab schedule and edit it with `crontab -e`.
- [ ] I know cron runs with a minimal environment and I use absolute paths + output redirection.
- [ ] I can locate and read system logs in `/var/log` and follow them with `tail -f`.
- [ ] I can do basic log analysis (grep/awk) for errors and status-code counts.
- [ ] I understand log rotation and why it prevents disk-full outages; I know containers log to stdout.

## Resources
- crontab.guru (schedule expression helper): https://crontab.guru/
- `man 5 crontab`, `man logrotate`
- systemd timers vs cron: https://wiki.archlinux.org/title/Systemd/Timers
- Linux logging overview: https://www.loggly.com/ultimate-guide/linux-logging-basics/
