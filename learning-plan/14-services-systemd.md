# 14 — Services & systemd

## Goals
- Understand what a "service" (daemon) is and how Linux supervises long-running software.
- Use `systemctl` to start/stop/enable services and `journalctl` to read their logs.
- Know the shape of a unit file so you can run *your own* app as a managed service.
- Understand the catch: containers often don't run systemd — and what to do about it.

## Concepts
- **What's a service/daemon?** A long-running background program that provides functionality: a web server (nginx), a database (postgres), the SSH server (sshd). It starts at boot, runs unattended, restarts on failure, and logs somewhere. Something has to *supervise* it — that's the **init system**.
- **systemd is the init system** on Debian/Ubuntu (and most modern distros). It's PID 1: it starts services in the right order, restarts crashed ones, tracks their processes (cgroups), and collects their logs. You drive it with `systemctl`.
- **`systemctl` — manage services:**
  - `systemctl status nginx` — is it running? recent log lines, PID, memory, enabled-at-boot status.
  - `systemctl start nginx` / `stop nginx` / `restart nginx` — control it now.
  - `systemctl reload nginx` — re-read config **without** dropping connections (where the service supports it — nginx does).
  - `systemctl enable nginx` — start it automatically at boot. `disable` — don't. `enable --now` = enable + start in one step.
  - `systemctl list-units --type=service` — everything running. `systemctl is-active nginx` / `is-enabled nginx` — scriptable yes/no.
- **`journalctl` — read logs collected by systemd:**
  - `journalctl -u nginx` — all log output from the nginx unit. `-u nginx -f` — follow live (like `tail -f`). `-u nginx -n 50` — last 50 lines.
  - `journalctl --since "10 min ago"`, `journalctl -p err` (errors only), `journalctl -b` (this boot). This is the first place you look when a service won't start.
- **Unit files — how a service is defined.** A `.service` file (in `/etc/systemd/system/` for your own, `/lib/systemd/system/` for packaged ones) describes how to run something:
  ```ini
  [Unit]
  Description=My web app
  After=network.target

  [Service]
  ExecStart=/usr/local/bin/myapp
  Restart=on-failure
  User=www-data
  WorkingDirectory=/srv/myapp

  [Install]
  WantedBy=multi-user.target
  ```
  After creating/editing one, run `systemctl daemon-reload` so systemd re-reads it, then `systemctl enable --now myapp`. This is exactly how you'd run a Go/Node binary as a proper service on a VM — `Restart=on-failure` gives you free crash-recovery.
- **The older `service` command** (`service nginx start`, `service nginx status`) still works as a thin wrapper and shows up in old docs — but `systemctl` is the real interface today.
- **The container catch (read carefully).** Most Docker base images **do not run systemd** — the container's PID 1 is your app or a shell, not `systemd`. So `systemctl`/`journalctl` may be missing or error with *"System has not been booted with systemd as init system."* This is by design: a container is meant to run **one** foreground process, and Docker itself is the supervisor (`restart:` policy in compose = systemd's `Restart=`). Ways to learn systemd anyway:
  1. **Concept + a real VM:** understand it here; practice fully on any cloud VM or a full Linux install — the commands are identical.
  2. **A systemd-enabled container:** images/flags exist that boot systemd as PID 1 (the practice README notes an optional variant). Heavier, but lets you run `systemctl` for real.
  3. **Map the ideas to Docker:** "enable at boot" ≈ `restart: unless-stopped`; "one unit = one service" ≈ "one container = one service"; "`journalctl -u x`" ≈ `docker compose logs x`. Recognizing this mapping is itself a key insight for Part 7.

## Exercises
1. Probe your environment first: run `systemctl status` (or `pidof systemd`). If you get the "not booted with systemd" message, note it — your container uses a plain init. That's expected; do the conceptual exercises and the Docker mapping.
2. (If systemd IS available, e.g. on a VM or systemd container) `systemctl list-units --type=service --state=running` — read what's running. `systemctl status ssh` if present.
3. Install nginx and inspect its packaged unit (the file exists even without systemd running): `apt install -y nginx`, then `cat /lib/systemd/system/nginx.service` and map each line to the concepts above.
4. Read logs (whichever applies): if systemd runs, `journalctl -u nginx -n 20`; otherwise read nginx's own log files `tail -n 20 /var/log/nginx/error.log` and note that *without* systemd a service logs to files, *with* systemd it can log to the journal.
5. Write a unit file (good practice even if you can't run it): create `/etc/systemd/system/hello.service` for a script that loops `date >> /tmp/hello.log; sleep 5`. Fill in `[Unit]/[Service]/[Install]`. Ask Claude to review it. If systemd is live: `systemctl daemon-reload && systemctl enable --now hello`, then watch `tail -f /tmp/hello.log` and `systemctl status hello`.
6. Map to Docker: in your practice `docker-compose.yml`, find (or add) a `restart:` policy and write down which systemd concept it replaces. List three more systemd↔Docker equivalents with Claude.

## Best Practices & Pitfalls
- **`reload` beats `restart` for live services.** `systemctl reload nginx` (or `nginx -s reload`) re-reads config with zero downtime; `restart` briefly drops connections. Reload after editing config; restart only when reload isn't supported or the binary changed.
- **Always `daemon-reload` after editing a unit file.** systemd caches unit definitions; without `systemctl daemon-reload` it keeps running the old version and your edits seem ignored.
- **`Restart=on-failure` + a non-root `User=` is the production default** for your own services: it auto-recovers from crashes and limits privilege. Don't run app services as root.
- **Check `journalctl -u <svc> -n 50` the moment a service won't start.** `systemctl status` shows a teaser; the journal has the real error (bad config path, port in use, permission denied).
- **Pitfall: expecting `systemctl` to work in a normal container.** It won't, and that's correct — don't try to bolt systemd into an app image. Use Docker's own supervision (restart policies, healthchecks) instead; that's the container-native equivalent (Part 7).
- **Pitfall: `enable` vs `start`.** `enable` = "at boot," `start` = "right now." Forgetting `enable` means the service silently doesn't come back after a reboot. `enable --now` does both.

## Checklist
- [ ] I can explain what a daemon is and that systemd (PID 1) supervises services.
- [ ] I can start/stop/restart/reload and enable/disable a service with `systemctl`.
- [ ] I can read a service's logs with `journalctl -u <svc>` (and follow with `-f`).
- [ ] I can read a `.service` unit file and write a simple one for my own app.
- [ ] I understand why containers usually lack systemd and how Docker's restart/healthcheck map to it.

## Resources
- systemd for administrators: https://www.freedesktop.org/wiki/Software/systemd/
- `man systemctl`, `man journalctl`, `man systemd.service`
- DigitalOcean systemd essentials: https://www.digitalocean.com/community/tutorials/systemd-essentials-working-with-services-units-and-the-journal
- Why containers don't run systemd: https://developers.redhat.com/blog/2019/04/24/how-to-run-systemd-in-a-container
