# 24 — Multi-Service Infra with Compose

## Goals
- Move from one growing server to **several services** orchestrated by Docker Compose.
- Understand Compose networking (service discovery by name), volumes, env/`.env`, and dependencies.
- Stand up a real shape: reverse proxy → app → database, all defined in one file.

## Concepts
- **Why multiple services now?** You've learned to run many things *inside one* Linux box (Steps 22–23). The modern alternative is **one service per container**, wired together by Compose. Benefits: each service scales/restarts/logs independently, uses its own language/runtime, and is isolated. Compose is the tool that declares the whole topology in one `docker-compose.yml` and brings it up with `docker compose up`.
- **Anatomy of a Compose file:**
  ```yaml
  services:
    proxy:
      image: nginx:1.27
      ports: ["8080:80"]              # host:container — publish to your Mac
      volumes:
        - ./proxy/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      depends_on: [api]

    api:
      build: ./api                    # build from a Dockerfile in ./api
      environment:
        - DATABASE_URL=postgres://app:secret@db:5432/app
      depends_on:
        db:
          condition: service_healthy  # wait until db's healthcheck passes

    db:
      image: postgres:16
      environment:
        - POSTGRES_USER=app
        - POSTGRES_PASSWORD=secret
        - POSTGRES_DB=app
      volumes:
        - db-data:/var/lib/postgresql/data   # named volume → data persists
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U app"]
        interval: 5s
        retries: 5

  volumes:
    db-data:
  ```
- **Networking & service discovery (the key idea).** Compose puts all services on a shared **bridge network** and runs an internal **DNS** server: each service is reachable from the others **by its service name** as a hostname. So the proxy reaches the API at `http://api:3000` and the API reaches Postgres at `db:5432` — *no IP addresses, no `localhost`*. This is the direct fix for the Step 23 pitfall: inside containers, `127.0.0.1` is the container itself; **use the service name** to reach another service.
- **`ports:` vs internal-only.** `ports: "8080:80"` *publishes* a service to your host (host port 8080 → container port 80). Services with **no** `ports:` are reachable only *inside* the Compose network — which is exactly what you want for the API and database (the database should **not** be published to your host). This is access control by topology, echoing Step 16: only the proxy needs a published port.
- **Volumes (two kinds, from Step 21):**
  - **Named volumes** (`db-data:`) — Docker-managed storage that persists across `down`/`up`; the right choice for **databases** and stateful data.
  - **Bind mounts** (`./proxy/nginx.conf:/etc/...:ro`) — map a host file/dir in, often read-only (`:ro`) for configs. Edit the config on your Mac, restart the proxy, done.
- **Configuration & secrets via env:** `environment:` sets variables inside a service (Step 18's env vars). Keep values out of the file with an **`.env`** file (Compose auto-loads it) or `env_file:`. **Never commit real secrets** — `.env` belongs in `.gitignore` (it already is in this repo). For production, use Docker/orchestrator secrets, not plaintext env.
- **Dependencies & startup order:** `depends_on` controls start *order*, but a container being "started" ≠ "ready." Use `depends_on` with `condition: service_healthy` plus a **`healthcheck`** so the API waits until Postgres actually accepts connections — otherwise the API races ahead and crashes on a refused connection. (App-side retry/`until` loops from Step 19 are the belt-and-suspenders complement.)
- **Driving the stack:** `docker compose up -d` (start all), `docker compose ps` (status + health), `docker compose logs -f api` (follow one service's logs — the container-native `tail -f`), `docker compose exec api bash` (shell into a service), `docker compose down` (stop/remove; add `-v` to also delete named volumes — careful, that wipes the DB), `docker compose up -d --build` (rebuild after Dockerfile/code changes), `docker compose up -d --scale api=3` (run 3 API instances).
- **Where this fits the growing server.** Your single lab box now becomes a small *system*: an nginx **proxy** container routing to an **api** container (the Step 23 routing, but `proxy_pass http://api:3000`), backed by a **db** container with a persistent volume. This is the template every real web stack follows.

## Exercises
1. Plan the topology on paper: proxy (published `8080:80`), api (internal only), db (internal only, named volume). Mark which services get a `ports:` entry and which don't, and why. Confirm your reasoning with Claude.
2. Build it incrementally in `practice/` (see the practice README's multi-service section). Start with just `proxy` + a static page, `docker compose up -d`, and `curl http://localhost:8080` from your Mac.
3. Add the `api` service (a tiny app — even `python3 -m http.server 3000` in a small Dockerfile, or a minimal Flask/Node app). Point the proxy's `location /api/` at `http://api:3000`. Reload/recreate and `curl http://localhost:8080/api/`. Note you used the **service name**, not an IP.
4. Prove service-name DNS: `docker compose exec proxy sh -c "getent hosts api"` and `curl http://api:3000` from *inside* the proxy container. Then try `curl http://127.0.0.1:3000` inside the proxy and watch it fail — cementing the Step 23 pitfall.
5. Add Postgres with a named volume + healthcheck. `docker compose up -d`, `docker compose ps` (see `healthy`), then write data: `docker compose exec db psql -U app -c "create table t(x int); insert into t values (1);"`. Run `docker compose down && up -d` and confirm the row survives (volume persistence). Then discuss what `down -v` would have done.
6. Env & `.env`: move the DB password into a `practice/.env` file, reference it via `${POSTGRES_PASSWORD}` in compose, confirm `.env` is gitignored, and verify the stack still comes up. 
7. Logs & scale: `docker compose logs -f proxy` while you curl it; then `docker compose up -d --scale api=2` and observe Compose start a second api instance (and how nginx upstream/Docker DNS spreads requests).

## Best Practices & Pitfalls
- **Reach other services by *service name*, never `localhost`/IP.** `db:5432`, `http://api:3000` — Compose DNS resolves these. Hardcoding `127.0.0.1` (means "this container") or a guessed IP (changes every run) is the most common multi-service failure.
- **Only publish what must be public.** Give `ports:` to the proxy alone; leave api/db internal. A database published to `0.0.0.0` on your host is a real security hole (Step 16). Topology *is* your firewall.
- **`depends_on` ≠ "ready" — add healthchecks.** Start order doesn't guarantee the dependency accepts connections. Use `condition: service_healthy` + a `healthcheck`, and have apps retry on startup. Otherwise you'll hit flaky "connection refused" on `up`.
- **Databases go in named volumes; configs come in as read-only bind mounts.** Never keep DB data in the container's writable layer (lost on recreate). Mount configs `:ro` so a container can't mutate them.
- **Keep secrets out of git.** Use `.env` (gitignored) for local, real secret management in production. Don't bake passwords into the image or commit them in `docker-compose.yml`.
- **Pitfall: `docker compose down -v` deletes your data.** The `-v` removes named volumes — great for a clean reset, catastrophic if you forget it wipes the database. Know exactly what `down`, `down -v`, and `up --build` each do before running them.
- **Pitfall: editing a bind-mounted config and not restarting the service.** The file changed but nginx/the app must reload/recreate to pick it up (`docker compose restart proxy` or `exec proxy nginx -s reload`).

## Checklist
- [ ] I can write a `docker-compose.yml` with multiple services, ports, volumes, and env.
- [ ] I reach other services by service name via Compose's internal DNS (not `localhost`/IP).
- [ ] I publish only the proxy and keep the api/db internal to the Compose network.
- [ ] I persist databases with named volumes and gate startup with `depends_on` + healthchecks.
- [ ] I can drive the stack: `up`/`down`/`logs`/`exec`/`--build`/`--scale`, and I know what `down -v` destroys.

## Resources
- Compose file reference: https://docs.docker.com/reference/compose-file/
- Compose networking & DNS: https://docs.docker.com/compose/how-tos/networking/
- Compose env / `.env`: https://docs.docker.com/compose/how-tos/environment-variables/
- Healthchecks & `depends_on`: https://docs.docker.com/reference/compose-file/services/#depends_on
