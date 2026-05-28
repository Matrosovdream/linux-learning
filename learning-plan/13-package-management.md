# 13 — Package Management

## Goals
- Install, update, and remove software the Debian/Ubuntu way with `apt`.
- Understand repositories, the package index, and `apt` vs `dpkg`.
- Install the tools you'll need for the rest of the course cleanly and reproducibly.

## Concepts
- **What a package manager does.** Instead of downloading random binaries, you ask `apt` for software by name; it fetches the right version *and its dependencies* from trusted **repositories**, installs them to standard locations (`/usr/bin`, configs in `/etc`, etc.), and tracks everything so it can update or remove it cleanly. This is the single biggest day-to-day difference from a Mac/`brew` workflow.
- **`apt` — the high-level tool (use this 95% of the time):**
  - `apt update` — refresh the **package index** (the catalog of what's available and at what version). Run this *before* installing — a fresh container's index may be empty/stale. It does **not** upgrade anything.
  - `apt upgrade` — install newer versions of already-installed packages. `apt full-upgrade` handles upgrades that need adding/removing packages.
  - `apt install <pkg>` — install a package + dependencies. `apt install -y` auto-confirms (essential in scripts/Dockerfiles). Install several at once: `apt install -y curl vim htop`.
  - `apt remove <pkg>` — uninstall but keep config files. `apt purge <pkg>` — uninstall **and** delete its config. `apt autoremove` — remove orphaned dependencies nothing needs anymore.
  - `apt search <term>` — find packages. `apt show <pkg>` — details (version, size, dependencies, description). `apt list --installed` — what's installed.
- **`dpkg` — the low-level tool** `apt` is built on. You reach for it occasionally:
  - `dpkg -l` — list installed packages. `dpkg -L <pkg>` — list the files a package installed (e.g. `dpkg -L nginx` shows every nginx path). `dpkg -S /path/to/file` — which package owns a file. `dpkg -i file.deb` — install a local `.deb` (then `apt -f install` to fix missing deps).
  - Rule of thumb: `apt` resolves dependencies and talks to repos; `dpkg` operates on individual `.deb` files and can't fetch dependencies itself.
- **Repositories & sources.** The list of repos lives in `/etc/apt/sources.list` and `/etc/apt/sources.list.d/*`. Each entry points to a server, a release (e.g. `bookworm` for Debian 12), and components (`main`, `contrib`, `non-free`). Packages are cryptographically signed; `apt` verifies signatures against keys it trusts — that's what makes installs secure.
- **Adding third-party repos** (e.g. official nginx or Docker repos) means adding their signing key and a source entry, then `apt update`. You'll do this when a distro's bundled version is too old. Prefer official repos over piping random install scripts into your shell.
- **The update-then-install habit:** `apt update && apt install -y <pkg>`. Skipping `apt update` is the #1 cause of "Unable to locate package" and "404" errors on a fresh system or container.
- **Docker relevance (big one):** every `RUN apt-get install` line in a Dockerfile is exactly this. The conventions for slim images: use `apt-get` (the stable scripting interface), combine `update`/`install` in one `RUN`, add `--no-install-recommends`, and `rm -rf /var/lib/apt/lists/*` afterward to shrink the image. We use these in Step 21. (`apt` is for humans; `apt-get`/`apt-cache` are the stable, scripting-safe commands.)

## Exercises
1. Refresh and inspect: `apt update`, then `apt list --upgradable`. Read what (if anything) is out of date.
2. Search & inspect before installing: `apt search htop`, then `apt show htop` — note its size and dependencies.
3. Install your toolkit: `apt install -y curl wget vim nano htop tree less net-tools iproute2 ca-certificates`. Verify a couple: `which curl`, `htop --version`.
4. See what a package put where: `dpkg -L htop | head`, then `dpkg -L tree`. Find which package owns a file: `dpkg -S /usr/bin/curl`.
5. Remove vs purge: `apt install -y cowsay`, run `cowsay hi`, then `apt remove -y cowsay` (binary gone, configs may remain). Reinstall and `apt purge -y cowsay`. Run `apt autoremove -y` to clean orphans.
6. Inspect repos: `cat /etc/apt/sources.list` (and `ls /etc/apt/sources.list.d/`). Identify the Debian release codename your container uses (cross-check with `cat /etc/os-release`).
7. Break and fix: try `apt install -y this-package-does-not-exist` and read the error. Then try installing *without* a prior `apt update` on a fresh container and observe the "Unable to locate" failure — internalize the update-first habit.

## Best Practices & Pitfalls
- **Always `apt update` before `apt install`.** Without a current index, `apt` doesn't know packages exist or where to fetch them — the classic "Unable to locate package" / "404 Not Found" errors. Make `apt update && apt install -y …` one reflex.
- **Use `purge` + `autoremove` to truly clean up.** `remove` leaves config files behind in `/etc`; `purge` removes them; `autoremove` clears dependencies nothing else needs. Otherwise cruft accumulates.
- **Prefer official repositories over `curl | bash` installers.** Repos are signed and updatable via `apt`; a piped install script runs arbitrary code as root and won't get security updates. When you must add a repo, add its signing key properly.
- **`apt` vs `apt-get` in scripts.** Interactive `apt` prints a warning that its CLI "is not stable." In Dockerfiles and scripts use `apt-get`/`apt-cache` — they're guaranteed stable. Add `-y` (and `DEBIAN_FRONTEND=noninteractive` for packages that ask questions).
- **Pitfall: ignoring held-back packages.** `apt upgrade` sometimes "keeps back" packages that need new/removed deps; use `apt full-upgrade` deliberately when appropriate (and understand what it'll change first).
- **Pitfall: installing the distro's ancient version.** Debian stable favors stability over freshness; if you need a current nginx/node/etc., add the official upstream repo rather than fighting the old packaged version.

## Checklist
- [ ] I run `apt update` before installing, and I understand it only refreshes the index.
- [ ] I can install, remove, purge, and `autoremove` packages.
- [ ] I can search/inspect packages (`apt search`/`show`) and list package files (`dpkg -L`).
- [ ] I understand repos and `/etc/apt/sources.list`, and why signed repos beat piped installers.
- [ ] I know why Dockerfiles use `apt-get … && rm -rf /var/lib/apt/lists/*`.

## Resources
- Debian apt guide: https://wiki.debian.org/Apt
- Ubuntu package management: https://ubuntu.com/server/docs/package-management
- `man apt`, `man apt-get`, `man dpkg`
- Docker best practices (apt section): https://docs.docker.com/build/building/best-practices/#apt-get
