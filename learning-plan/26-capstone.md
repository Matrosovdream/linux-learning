# 26 — Capstone: Build Your Web Infra

## Goals
- Bring the whole course together: stand up a small but realistic web stack on Linux, in Docker, from scratch.
- Apply everything — filesystem, permissions, users, processes, networking, ports, nginx, reverse proxy, Compose, persistence, logs.
- Be able to explain *every* layer of what you built and debug it when it breaks.

## The project
Build a **reverse-proxy-fronted multi-service web stack** as your growing server's final form:

```
                 ┌────────────────────────────────────────────┐
   you (curl) ──▶│  proxy (nginx)   published :8080 → :80      │
                 │     /        → web   (static or app UI)     │
                 │     /api/    → api   (an app service)       │
                 │     /auth/   → auth  (an app service)       │
                 └───────┬───────────────┬───────────────┬─────┘
                         │ internal net  │               │
                    ┌────▼───┐      ┌─────▼───┐     ┌─────▼────┐
                    │  web   │      │   api   │────▶│   db     │
                    │        │      │         │     │ (postgres│
                    └────────┘      └─────────┘     │  + volume)│
                                                    └──────────┘
```

- **proxy** — the only published service; routes by path; terminates TLS (self-signed for local); logs all requests.
- **web** — a static site or simple UI (start static, from Step 22).
- **api**, **auth** — small backend services (any language; even `python3 -m http.server`/Flask/Node stubs are fine — the *infra* is the point). `api` talks to `db` and may call `auth` internally.
- **db** — Postgres with a **named volume** so data persists; **not** published.
- Internal-only networking by **service name**; healthchecks gate startup; everything logs to stdout.

> Scope it to your level. A "stub" version (static web + two `http.server` backends + Postgres) exercises 100% of the Linux/infra skills. Swap in real apps later if you like — the surrounding infra doesn't change.

## Build it step by step
1. **Lay out the project** under `practice/` (see the practice README's capstone section): a folder per service, each with its `Dockerfile`/config, plus `docker-compose.yml` and a gitignored `.env`. Use what you learned about the filesystem to keep it tidy.
2. **db first.** Add Postgres with `POSTGRES_*` env from `.env`, a named volume for `/var/lib/postgresql/data`, and a `pg_isready` healthcheck. `docker compose up -d db`; confirm `healthy`; create a table and a row.
3. **api + auth.** Containerize each (small Dockerfile; `apt` clean-up habits from Step 13/21). Configure `api` with `DATABASE_URL` and `AUTH_URL=http://auth:8000` via env. Internal only — no `ports:`. Add `/healthz` endpoints + healthchecks. Bring them up; from inside `api`, `curl http://auth:8000/...` and connect to `db:5432` to prove service-name networking.
4. **proxy.** nginx container, bind-mount your config read-only, publish `8080:80` (and `8443:443` for TLS). `location` blocks route `/`→web, `/api/`→`http://api:...`, `/auth/`→`http://auth:...` with the required `X-Forwarded-*`/`Host` headers. `nginx -t` clean, then up.
5. **web.** Static files served by the proxy (or its own tiny container). Set correct `www-data` ownership/permissions (Step 07) if served from disk.
6. **TLS (stretch).** Self-signed cert, `listen 443 ssl`, key file `600`, redirect 80→443. `curl -k https://localhost:8443/`.
7. **Verify end-to-end** from your Mac: `curl http://localhost:8080/`, `/api/`, `/auth/` all return correctly; `db` is unreachable from your host (good); data survives `docker compose down && up -d`.

## Verification checklist (prove it works)
- [ ] `docker compose ps` shows every service `Up`/`healthy`.
- [ ] `curl http://localhost:8080/`, `/api/`, `/auth/` each reach the right service through the proxy.
- [ ] Only the proxy has a published port; `db`/`api`/`auth` are internal (confirm a host-side `curl` to the DB port fails).
- [ ] Services reach each other by **name** (`api`→`db:5432`, `api`→`auth:8000`); no hardcoded IPs/`localhost`.
- [ ] DB data persists across `docker compose down && up -d` (named volume), and you know `down -v` would wipe it.
- [ ] Killing a backend yields a `502` through the proxy and recovers when it returns; logs show why.
- [ ] No secrets are committed (`.env` is gitignored); configs are bind-mounted read-only.
- [ ] You can `docker compose logs -f <svc>` and trace a single request across the proxy and a backend.

## Going further (optional)
- **Scale** a stateless service (`--scale api=3`) and load-balance via nginx `upstream`/Docker DNS.
- **Network segmentation:** separate `frontend`/`backend` Docker networks so `db` is reachable only by `api`/`auth`.
- **Observability:** add a request-ID header at the proxy and log it everywhere; try a log-aggregation or metrics container.
- **A message broker** (Redis/RabbitMQ) for async communication between `api` and a worker service.
- **Real Let's Encrypt TLS** and an 80→443 redirect on a real domain/VM.
- **Deploy for real:** provision a cloud VM, harden SSH (Step 17), set up `ufw` (Step 16), install Docker, and run this same Compose stack — the skills transfer 1:1.
- **Toward orchestration:** read how this maps to Kubernetes (Deployments=services, Services=internal DNS, Ingress=your proxy).

## Best Practices & Pitfalls (the course in one list)
- **Least privilege everywhere:** non-root service users, `600` secrets/keys, `755`/`644` web files, publish only the edge, segment networks.
- **Service name, not `localhost`/IP**, for inter-service calls — the single most common multi-container bug.
- **Persist state in volumes; keep services stateless.** Know what `down` vs `down -v` destroys *before* you run it.
- **Test config before reloading** (`nginx -t`), **reload over restart** for zero downtime, and **read the logs** — the status code (403/404/502/504) tells you which layer to fix.
- **Design for failure:** healthchecks, timeouts, retries/backoff, restart policies. A dependency *will* be momentarily down.
- **Keep secrets out of git; configs read-only; images small and clean.**
- **Right-size the architecture:** this multi-service stack is great practice, but for a small real app a clean monolith on one server may be the better engineering choice. Know *why* you chose what you built.

## Wrap-up
You started not knowing what the shell was. You can now navigate and administer a Linux system, manage users/permissions/processes/packages/services, work the network and ports, write scripts, and stand up and route a real multi-service web stack in Docker — and explain every layer. Update [PROGRESS.md](PROGRESS.md), and revisit any step that still feels shaky: mastery comes from repeating the exercises until the commands are reflex.

Ask Claude for a final review of your capstone: have it poke holes in your security, networking, and failure-handling, then iterate.

## Resources
- Compose production notes: https://docs.docker.com/compose/how-tos/production/
- nginx full admin guide: https://docs.nginx.com/nginx/admin-guide/
- Deploy Docker on a VM: https://docs.docker.com/engine/install/debian/
- Kubernetes basics (next step after this): https://kubernetes.io/docs/tutorials/kubernetes-basics/
