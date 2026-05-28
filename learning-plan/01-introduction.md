# 01 — Introduction to Linux

## Goals
- Understand what "Linux" actually is — and the difference between the kernel, a distribution, and the shell.
- Know why Linux runs almost all web servers, containers, and cloud infrastructure.
- Build a mental model of the pieces (kernel, userland, shell, package manager) before touching a command.

## Concepts
- **What is Linux?** Strictly, Linux is just the **kernel** — the core program that talks to hardware, manages memory and CPU time, and exposes files, processes, and networking. Everything else you "use" (the shell, `ls`, `nginx`, Python) is separate software bundled around it.
- **Kernel vs distribution.** A **distribution** (distro) = the Linux kernel **+** a userland (GNU coreutils, a shell, libraries) **+** a **package manager** + sensible defaults. Examples: **Debian** and **Ubuntu** (our focus, use `apt`), Fedora/RHEL (use `dnf`/`rpm`), Alpine (tiny, uses `apk`, common in Docker). Same kernel, different packaging and conventions.
- **Why web developers must know Linux:**
  - Almost every production web server, container, and cloud VM runs Linux.
  - **Docker containers are Linux** — an image like `nginx` or `node` is a slice of a Linux userland. Knowing Linux *is* knowing what's inside your containers.
  - Deploying, debugging, reading logs, fixing permissions, opening ports, and wiring services together are all Linux skills.
- **The shell.** The **shell** is the program that reads your typed commands and runs them. The default on Debian/Ubuntu is **bash**. The shell is *not* the kernel — it's a normal program that asks the kernel to do things on your behalf. You'll live in the shell for this entire course.
- **Everything is a file.** A defining Linux idea: files, directories, devices, running processes, even network info are exposed as paths you can read/write. This is why the same handful of tools (`cat`, `ls`, `>` ) work on almost everything.
- **Multi-user from the ground up.** Linux assumes many users and enforces **permissions** and **ownership** on every file and process. The all-powerful user is **root**. Web servers run as limited users on purpose — a compromised service shouldn't own the box.
- **Open source.** The kernel and most userland are open source. Practically, this means abundant documentation, the ability to inspect anything, and no licensing barrier to spinning up as many servers/containers as you like.
- **Linux vs macOS/Windows for you:** macOS is Unix-like (many commands overlap, which is why your Mac terminal feels similar), but it is *not* Linux — package management, init system, and paths differ. That's exactly why we practice on a real Linux box in Docker rather than on macOS directly.

## Exercises
1. In your own words, write one sentence each for: **kernel**, **distribution**, **shell**, **package manager**. Ask Claude to check them.
2. List three pieces of software you already use that run on Linux (hint: think about the websites and containers you touch). Ask Claude which ones are "Linux all the way down."
3. Look at the Docker images you already have (`docker images` on your Mac) and pick one — ask Claude which Linux distro it's likely based on and how to tell.
4. Read the "why everything is a file" idea again and predict: if a process is "a file", what might you be able to do with it? (We'll confirm in Part 4.)

## Best Practices & Pitfalls
- **Don't conflate the terminal app, the shell, and Linux.** The terminal is a window; the shell (bash) is the program inside it; Linux is the OS underneath. Keeping these separate will save you confusion later.
- **Pitfall: assuming macOS == Linux.** Commands look alike, but `brew` isn't `apt`, macOS has no `systemd`, and some flags differ. Always practice on the Linux container, not the Mac shell.
- **Pitfall: fearing the command line.** It feels unforgiving at first, but it's just text in, text out. In the container you can break anything and rebuild in seconds — lean into that.
- **Get comfortable with "I'll look it up."** Nobody memorizes every command. The skill is knowing a tool *exists* and how to read its `--help`/`man` page (next lessons).

## Checklist
- [ ] I can explain that Linux (strictly) is the kernel, and a distro adds userland + package manager.
- [ ] I can name our target distro family (Debian/Ubuntu) and its package manager (`apt`).
- [ ] I understand the shell is a program that runs my commands, not the OS itself.
- [ ] I can explain why "Docker containers are Linux" matters for a web developer.
- [ ] I know what "everything is a file" and "root" mean at a high level.

## Resources
- What is the Linux kernel: https://www.kernel.org/
- Debian "What is Debian": https://www.debian.org/intro/about
- Ubuntu Server docs: https://ubuntu.com/server/docs
- The Linux Documentation Project (classic, still useful): https://tldp.org/
- "The Unix philosophy" overview: https://en.wikipedia.org/wiki/Unix_philosophy
