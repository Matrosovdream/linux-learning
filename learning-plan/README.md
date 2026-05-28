# Linux Learning Plan

A step-by-step path to learn Linux from zero to building real web infrastructure. Each step is a self-contained lesson with goals, concepts, exercises, best-practice notes, and resources. Claude is your tutor — ask questions as you go.

The whole course is hands-on. You'll do every exercise on a **real Linux server running inside Docker** — see [../practice/README.md](../practice/README.md). The philosophy: read a little theory, then *type the commands yourself*, repeatedly, until they're second nature.

## How to use this plan

1. Work through steps in order (01 → 26).
2. Start your lab once (see [practice/README.md](../practice/README.md)) and keep a shell open in it.
3. For each step:
   - Read the lesson file (`NN-title.md`).
   - **Type every command** in your lab. Don't copy-paste — typing builds memory. Break things on purpose and fix them.
   - Do the exercises. Many ask you to add something to your growing server in `/practice`.
   - Ask Claude to explain anything unclear, or to review what you built.
4. When you finish a step, update [PROGRESS.md](PROGRESS.md) — that's where we track where you are.

Every lesson has a **Best Practices & Pitfalls** section and a **Checklist**. Read them even in a hurry — sysadmin skill is the accumulation of these small habits.

## What we're targeting

- **Distro:** Debian / Ubuntu family (`apt`, `dpkg`, `systemd`) — the most common base for web servers and Docker images.
- **Where it runs:** a single Debian container you start with Docker Compose — your "growing server."
- **Orientation:** web development. We bias every topic toward what you actually need to run web services: the filesystem, permissions, processes, networking, ports, SSH, web servers, reverse proxies, and multi-service / microservice infra.
- **End goal:** comfortably stand up and route a small web stack (reverse proxy → several services → database) on Linux, and understand every layer underneath it.

> **Note:** Nothing is installed on *your* machine except Docker (which you already have). The Linux you learn on is the container — break it freely, throw it away, rebuild it.

## Steps

### Part 1 — Foundations
- [01 — Introduction to Linux](01-introduction.md) — what Linux is, kernel vs distro, why it runs the web
- [02 — Environment Setup: Your Linux Lab](02-environment-setup.md) — start a Debian server in Docker, get a shell, persist work
- [03 — The Shell & Command Line Basics](03-the-shell.md) — bash, command anatomy, `--help`, `man`, history, tab-completion

### Part 2 — Filesystem & Files
- [04 — The Filesystem Hierarchy](04-filesystem-hierarchy.md) — `/etc`, `/var`, `/usr`, `/home`, `/proc`, where web stuff lives
- [05 — Navigating & Managing Files](05-files-navigation.md) — `ls cd pwd cp mv rm mkdir ln`, paths, `tree`
- [06 — Viewing & Editing Files](06-viewing-editing.md) — `cat less head tail`, `nano`, surviving `vim`
- [07 — Permissions & Ownership](07-permissions.md) — `rwx`, `chmod`, `chown`, users/groups, `sudo`, why web servers care

### Part 3 — Text, Streams & Search
- [08 — Redirection, Pipes & Streams](08-redirection-pipes.md) — stdin/stdout/stderr, `> >> | tee`, exit codes
- [09 — Text Processing](09-text-processing.md) — `grep sed awk cut sort uniq wc tr` — reading logs and configs
- [10 — Finding Things](10-search-glob.md) — `find`, globbing, `locate`, a working knowledge of regex

### Part 4 — Processes, Users & System
- [11 — Processes, Jobs & Signals](11-processes.md) — `ps top htop kill`, foreground/background, signals, `nohup`
- [12 — Users, Groups & sudo](12-users-groups.md) — `/etc/passwd`, `useradd`, `su`, `sudo`, service users
- [13 — Package Management](13-package-management.md) — `apt`, `dpkg`, repositories, updates, installing your stack
- [14 — Services & systemd](14-services-systemd.md) — `systemctl`, `journalctl`, units, running software as a service

### Part 5 — Networking (the web-dev core)
- [15 — Networking Fundamentals](15-networking-fundamentals.md) — IPs, ports, DNS, `ip`, `ping`, `curl`, `wget`
- [16 — Ports, Sockets & Firewall](16-ports-firewall.md) — `ss`, listening ports, `ufw`/`iptables`, what's exposed
- [17 — SSH & Remote Access](17-ssh.md) — `ssh`, key pairs, `scp`/`rsync`, `~/.ssh/config`, hardening

### Part 6 — Shell Scripting & Automation
- [18 — Environment, PATH & Shell Config](18-environment-shell.md) — env vars, `PATH`, `.bashrc`/`.profile`, aliases
- [19 — Bash Scripting](19-bash-scripting.md) — variables, conditionals, loops, functions, args, `set -euo pipefail`
- [20 — Scheduling & Logs](20-cron-logs.md) — `cron`, `/var/log`, `logrotate`, keeping a server tidy

### Part 7 — Web Infrastructure & Docker
- [21 — Linux in Docker](21-linux-in-docker.md) — images vs containers, why Linux is the base, Dockerfiles, the lab explained
- [22 — Running a Web Server](22-web-server.md) — install & run nginx, serve static files, read its config layout
- [23 — Reverse Proxy & Routing](23-reverse-proxy-routing.md) — nginx as a reverse proxy, `location` routes, upstreams, TLS basics
- [24 — Multi-Service Infra with Compose](24-multi-service-compose.md) — many services, Docker networks, volumes, env, dependencies
- [25 — Microservices on Linux](25-microservices.md) — service-per-container, internal DNS, config, health checks, scaling
- [26 — Capstone: Build Your Web Infra](26-capstone.md) — reverse proxy → multiple app services → database, with logs and health checks

## Progress

See [PROGRESS.md](PROGRESS.md) for the current step and notes from past lessons.
