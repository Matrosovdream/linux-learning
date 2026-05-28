# 21 — Linux in Docker

## Goals
- Understand what a container *is* in Linux terms — and why "a container is just Linux processes."
- Read and write a Dockerfile; understand images, layers, and the build cache.
- Fully understand the practice lab's `Dockerfile` and `docker-compose.yml` you've been using.

## Concepts
- **A container is not a VM.** It's a normal Linux **process** (or process tree) that the kernel **isolates** using two kernel features: **namespaces** (give the process its own view of the filesystem, network, PIDs, hostname, users) and **cgroups** (limit its CPU/memory). There's no second kernel — containers share the host's Linux kernel. This is why containers are lightweight and start in milliseconds, and why **everything you learned in Steps 01–20 applies inside a container**: it *is* Linux.
- **Image vs container (precise version):**
  - An **image** is a read-only, layered filesystem snapshot + metadata (which command to run, env, exposed ports). Think "a frozen Linux userland plus your app."
  - A **container** is a running (or stopped) instance of an image, with a thin writable layer on top. Many containers can run from one image. Delete the container, the image remains.
- **Layers & the build cache.** Each instruction in a `Dockerfile` creates a **layer**. Docker caches layers and reuses them if nothing above changed — so ordering matters: put rarely-changing steps (install OS packages) *before* frequently-changing ones (copy your app code), or you bust the cache and rebuild everything each time. This is why `COPY package.json` + `install` comes before `COPY . .` in real Dockerfiles.
- **Dockerfile essentials:**
  - `FROM debian:12-slim` — the base image (a slim Debian userland). Everything builds on this.
  - `RUN <cmd>` — run a command at *build* time (install packages, create dirs). Each `RUN` is a layer.
  - `COPY src dst` — copy files from your project into the image.
  - `WORKDIR /app` — set the working directory for later instructions and at runtime.
  - `ENV KEY=value` — set an environment variable baked into the image.
  - `EXPOSE 80` — document a port (doesn't publish it — that's compose's `ports:`).
  - `CMD ["nginx","-g","daemon off;"]` / `ENTRYPOINT` — the process that runs as **PID 1** when the container starts. A container lives exactly as long as this process does (key insight: the container stops when PID 1 exits — which is why our lab runs a long-lived process to stay up).
  - **apt in Dockerfiles** (from Step 13): `RUN apt-get update && apt-get install -y --no-install-recommends <pkgs> && rm -rf /var/lib/apt/lists/*` — combine into one layer and clean the package index to keep the image small.
- **`docker` vs `docker compose`.** `docker build`/`run`/`ps`/`exec`/`logs` operate on single images/containers; **Compose** reads a `docker-compose.yml` describing one or more services (image or build, ports, volumes, env, networks) so you start the whole thing with `docker compose up`. The lab uses Compose so settings are recorded, not retyped.
- **Volumes & bind mounts (why your `/work` persists):** containers are ephemeral — the writable layer dies with the container. **Volumes** and **bind mounts** map storage that outlives the container. A **bind mount** (`./server-data:/work`) maps a host folder into the container — that's how your lab files survive `down`/`up` and are editable from both your Mac and the container. Step 24 covers named volumes for databases.
- **Container-native equivalents of what you learned:** "run a service at boot" → `restart:` policy; "is it healthy?" → `healthcheck`; "service logs" → `docker compose logs`; "firewall/which ports are open" → `ports:` you publish; "one service per machine" → one service per container. Recognizing these mappings is the bridge from sysadmin to modern infra.

## Exercises
1. Read your lab files line by line: open `practice/Dockerfile` and `practice/docker-compose.yml`. For *every* line, say what it does (use the concepts above; ask Claude about anything unclear). This is the payoff — you now understand the environment you've used all course.
2. Inspect images & layers: on your Mac, `docker images` (see your lab image and its size), `docker compose images`, and `docker history <image>` (see the layers and which instruction made each).
3. See "a container is Linux processes": inside the lab run `ps aux` and note PID 1 is the lab's long-running command, not `systemd`. On your Mac, `docker compose top` shows the same processes from the host's view.
4. Prove the cache: add a harmless `RUN echo "layer test"` near the *end* of the Dockerfile, `docker compose build`, and watch which steps say `CACHED`. Move it near the *top* and rebuild — notice everything after it rebuilds. Remove it.
5. Edit-and-rebuild loop: add `tree` (or another tool) to the `apt-get install` line, `docker compose up -d --build`, shell in, and confirm the new tool exists. This is the real workflow for evolving a server image.
6. PID-1 lifetime: discuss with Claude why a container whose `CMD` is `bash` (non-interactive) exits immediately, and what trick the lab uses to stay running.
7. Bind mount reality: create a file in `/work` from inside the container, then find it in `practice/server-data/` on your Mac (and vice versa). Explain to yourself why this survives `docker compose down`.

## Best Practices & Pitfalls
- **Order Dockerfile steps from least- to most-frequently-changing** to exploit the layer cache. Installing packages every code change wastes minutes; structuring layers well makes rebuilds near-instant.
- **One concern per container.** A container should run one main process (one service). Don't cram nginx + app + database into one image "to keep it simple" — that fights the model and breaks scaling, logging, and restarts. Compose multiple containers instead (Steps 24–25).
- **Keep images small and clean:** use `-slim` bases, `--no-install-recommends`, and `rm -rf /var/lib/apt/lists/*`. Smaller images build/pull faster and have less attack surface.
- **The container stops when PID 1 exits.** If your container "won't stay up," check what its `CMD`/`ENTRYPOINT` is — a foreground, long-lived process keeps it alive; a command that returns immediately ends it.
- **Pitfall: storing important data only in the container's writable layer.** It's deleted with the container. Databases and anything you must keep go in a volume/bind mount (Step 24), never the ephemeral layer.
- **Pitfall: confusing `EXPOSE` with publishing.** `EXPOSE` is documentation; you actually reach a port only if compose `ports:` maps it (or another container shares its network). Step 16's "listening vs reachable" applies directly.

## Checklist
- [ ] I can explain a container as isolated Linux processes (namespaces + cgroups), not a VM.
- [ ] I can distinguish image vs container and explain layers and the build cache.
- [ ] I can read/write a basic Dockerfile (`FROM/RUN/COPY/WORKDIR/ENV/CMD`).
- [ ] I understand PID 1 and why a container lives only as long as its main process.
- [ ] I can explain how bind mounts/volumes make data persist, and I understand my lab files top to bottom.

## Resources
- Dockerfile reference: https://docs.docker.com/reference/dockerfile/
- Docker overview / what is a container: https://docs.docker.com/get-started/docker-overview/
- Dockerfile best practices: https://docs.docker.com/build/building/best-practices/
- Namespaces & cgroups (the Linux features behind containers): https://man7.org/linux/man-pages/man7/namespaces.7.html
