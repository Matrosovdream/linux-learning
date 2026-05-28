# linux-learning

A personal Linux course, learned from scratch and oriented toward web developers: the command line, how a Linux system is laid out, processes, networking, and building real web infrastructure (web servers, reverse proxies, multi-service and microservice setups) — all practiced on a real Linux box running inside Docker.

- **Plan & lessons:** [learning-plan/README.md](learning-plan/README.md)
- **Where I am right now:** [learning-plan/PROGRESS.md](learning-plan/PROGRESS.md)
- **Hands-on practice (your Linux lab):** [practice/README.md](practice/README.md)

## The idea

Two things happen in parallel:

1. **Theory + commands** — each numbered lesson in `learning-plan/` explains a topic and gives you commands to run and exercises to do. Read it, then *type every command yourself* in your lab.
2. **One growing server** — in `practice/` you run a single Debian Linux server inside Docker and build it up over the course: a shell to live in, then a web server, then a reverse proxy routing to several services, ending in a small multi-service web stack. You repeat the instructions until they're muscle memory.

Claude is your tutor — ask questions as you go, and ask for a review of anything you build.
