# 02 — Environment Setup: Your Linux Lab

## Goals
- Start a real Debian Linux server inside Docker — your "growing server" for the whole course.
- Get an interactive shell inside it and understand what just happened.
- Make your work **persist** so the container can be rebuilt without losing files.
- Learn the handful of `docker`/`docker compose` commands you'll use to drive the lab.

## Concepts
- **Why Docker for learning Linux?** You already have Docker. A container gives you a disposable, real Linux system in seconds — break it, delete it, rebuild it, with zero risk to your Mac. It's also exactly the environment your web apps deploy into, so the skills transfer directly.
- **Image vs container.** An **image** is a read-only template (e.g. `debian:12`). A **container** is a running instance of an image. We define our image in a `Dockerfile` and run it via **Docker Compose** (a `docker-compose.yml` that records the settings so you don't retype long `docker run` flags). Full detail is in Step 21 — for now, treat it as "this file describes my server."
- **The lab lives in [`practice/`](../practice/README.md).** It contains a `Dockerfile` (a Debian base + a few tools so the box isn't bare) and a `docker-compose.yml` (names the server, mounts a folder for persistence, maps ports for later web lessons).
- **Getting a shell.** `docker compose exec <service> bash` opens an interactive **bash** shell *inside* the running container. Your prompt changes to something like `root@linux-lab:/#` — you are now "in Linux."
- **Persistence with a volume.** By default, changes inside a container vanish when it's removed. The lab mounts a host folder (e.g. `practice/server-data/`) into the container so files you create there survive rebuilds. Anything *outside* that path is ephemeral — which is great for experiments.
- **The core lab commands** (run from inside `practice/`):
  - `docker compose up -d` — build (if needed) and start the server in the background.
  - `docker compose exec server bash` — open a shell inside it. (`server` is the service name.)
  - `docker compose ps` — see whether it's running.
  - `docker compose stop` / `start` — pause/resume without deleting.
  - `docker compose down` — stop and remove the container (volume-backed files persist).
  - `docker compose up -d --build` — rebuild the image after you change the `Dockerfile`.
  - `exit` (inside the container) — leave the shell; the server keeps running.
- **You'll often have two shells open:** one on your Mac (to run `docker compose ...`) and one *inside* the container (your Linux prompt). Knowing which is which is the first real skill — the prompt tells you.

## Exercises
1. Open a terminal on your Mac and `cd` into `practice/`. Read [`practice/README.md`](../practice/README.md) once, top to bottom.
2. Start the lab: `docker compose up -d`. Then `docker compose ps` — confirm the `server` is `Up`.
3. Get a shell: `docker compose exec server bash`. Note your new prompt. Run `whoami`, `hostname`, and `cat /etc/os-release`. Ask Claude to explain each line of `/etc/os-release`.
4. Create a file in the persistent area: `echo "hello linux" > /work/notes.txt`, then `cat /work/notes.txt`. (Path may differ — check the practice README.)
5. Test persistence: `exit`, then `docker compose down`, then `docker compose up -d`, shell back in, and `cat /work/notes.txt`. Did it survive? Now create a file *outside* `/work` (e.g. `/tmp/scratch.txt`), rebuild, and confirm it's gone. Ask Claude why.
6. Practice the loop: exit and re-enter the shell three times until starting/stopping/entering feels automatic.

## Best Practices & Pitfalls
- **Know which shell you're in.** A Mac prompt (`stan@macbook ~ %`) runs `docker` commands; a container prompt (`root@linux-lab:/#`) runs Linux commands. Running `apt` on your Mac or `docker` inside the container is the #1 beginner mix-up.
- **Keep real work in the persistent path.** Treat everything else as scratch. When in doubt, `cd /work` (or whatever the README defines) before creating files you care about.
- **`down` is safe here; it does not wipe your mounted folder.** But it *does* remove the container — anything written outside the volume is lost. That's a feature: it lets you reset a messed-up box instantly.
- **Rebuild after editing the `Dockerfile`.** Changes to the image (new packages, etc.) only take effect after `docker compose up -d --build`.
- **Pitfall: editing files on the Mac vs in the container.** The mounted folder is visible from *both* sides. Editing `practice/server-data/notes.txt` in VS Code and `cat /work/notes.txt` in the container show the same file — that's intentional and very handy.

## Checklist
- [ ] `docker compose up -d` starts my server and `docker compose ps` shows it `Up`.
- [ ] I can open and exit a bash shell inside the container.
- [ ] I can tell a Mac prompt from a container prompt at a glance.
- [ ] Files in the persistent path survive `down` + `up`; files elsewhere don't.
- [ ] I know how to rebuild the image after changing the `Dockerfile`.

## Resources
- Practice lab setup: [../practice/README.md](../practice/README.md)
- Docker Compose CLI: https://docs.docker.com/compose/reference/
- `docker compose exec`: https://docs.docker.com/reference/cli/docker/compose/exec/
- Debian official image (what our base is): https://hub.docker.com/_/debian
- `/etc/os-release` format: https://www.freedesktop.org/software/systemd/man/os-release.html
