# Practice — Your Linux Lab

This is where you *do* the course. It's a single **Debian Linux server running in Docker** that you grow over the 26 lessons: first a shell to live in, then a web server, then a reverse proxy routing to several services, ending in a small multi-service stack. Break it freely — it's disposable.

> You only need **Docker** installed on your Mac (you already have it). Nothing else is installed on your machine — the Linux you learn on is the container.

## Quick start

From this `practice/` folder:

```bash
docker compose up -d            # build + start the server in the background
docker compose exec server bash # open a shell INSIDE the Linux server
```

Your prompt becomes something like `root@linux-lab:/work#` — you're now "in Linux." Type `exit` to leave the shell (the server keeps running).

## The commands you'll use to drive the lab

Run these on your **Mac** (not inside the container):

| Command | What it does |
|---|---|
| `docker compose up -d` | Build (if needed) and start the server in the background |
| `docker compose exec server bash` | Open an interactive shell inside the server |
| `docker compose ps` | Is it running? |
| `docker compose stop` / `start` | Pause / resume without deleting |
| `docker compose down` | Stop & remove the container (**your `/work` files persist**) |
| `docker compose up -d --build` | Rebuild after you edit the `Dockerfile` |
| `docker compose logs -f server` | Follow the container's output |

**Two prompts, know the difference** (this is the #1 beginner mix-up — see lesson 02):
- Mac prompt (`stan@mac %`) → run `docker compose …` here.
- Container prompt (`root@linux-lab:/work#`) → run Linux commands (`ls`, `apt`, `nginx`) here.

## What persists, what doesn't

- **`/work` inside the container** is bind-mounted to **`./server-data/` on your Mac**. Files here **survive** `docker compose down` and are editable from *both* sides (open `practice/server-data/` in your editor, `cat /work/...` in the container — same files). Keep real work here.
- **Everything else** (installed packages, files outside `/work`, the home dir) lives in the container's writable layer and is **wiped** when you `docker compose down`. That's intentional — it lets you reset a messed-up box instantly.
- To make a tool or change **permanent across rebuilds**, add it to the [`Dockerfile`](Dockerfile) and `docker compose up -d --build`. This "edit Dockerfile → rebuild" loop is exactly how you evolve a real server image (lesson 21).

## Ports (for the web lessons, Part 7)

The compose file publishes two ports so you can reach web services from your Mac's browser/`curl`:

- `http://localhost:1010` → container port **80** (nginx, lesson 22+)
- `https://localhost:1011` → container port **443** (TLS, lesson 23)

## How the lab grows, lesson by lesson

You don't build everything at once — you add to this one server as you learn:

1. **Lessons 01–20 (foundations):** live in the shell. Do every exercise here. Install tools with `apt` as lessons ask. Keep notes and scripts in `/work`.
2. **Lessons 22–23 (web server & routing):** `apt install nginx` inside the server, serve a static site, then turn nginx into a reverse proxy in front of small backend apps — all inside this one container.
3. **Lessons 24–26 (multi-service & capstone):** evolve into a real *system*. Here you'll author a **new, multi-service `docker-compose.yml`** (a proxy container + app services + a Postgres container with a named volume). The lessons walk you through it; sketch your services first, then build incrementally.
   - Tip: keep the capstone stack in its own subfolder (e.g. `practice/stack/`) with a folder per service, its own `docker-compose.yml`, and a gitignored `.env` (copy [`.env.example`](.env.example)). Your foundational server above stays as-is for command practice.

## Repeat until it's reflex

The point of this folder is repetition. Re-run the exercises, break things on purpose (wrong permissions, killed processes, bad nginx config), read the error, and fix them. When a lesson's commands feel automatic, update [`../learning-plan/PROGRESS.md`](../learning-plan/PROGRESS.md) and move on. Ask Claude to review anything you build or to invent extra drills.

## Optional: practicing systemd

Most containers (including this one) don't run `systemd` as PID 1, so `systemctl`/`journalctl` won't work here — that's normal and explained in [lesson 14](../learning-plan/14-services-systemd.md). To practice systemd for real, use any cloud VM or a full Linux install (the commands are identical), or ask Claude about running a systemd-enabled container variant.

## Troubleshooting

- **`docker compose` says "no configuration file"** — you're not in the `practice/` folder. `cd` here first.
- **Shell exits immediately / container keeps restarting** — check `docker compose logs server`; the base image runs `sleep infinity` as PID 1 to stay alive (lesson 21).
- **Port 1010 already in use on your Mac** — something else is using it; change the host side in `docker-compose.yml` (e.g. `"1012:80"`). Find the culprit with `lsof -nP -iTCP:1010 -sTCP:LISTEN`.
- **A tool I installed vanished after `down`** — expected; add it to the `Dockerfile` and rebuild to make it permanent (see "What persists" above).
