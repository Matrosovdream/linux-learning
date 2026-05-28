# 25 — Microservices on Linux

## Goals
- Understand the microservice model and the Linux/Docker concepts that make it work.
- Wire several independent services together: internal networking, config, health, and scaling.
- Know the trade-offs — when microservices help and when one server is the better call.

## Concepts
- **What "microservices" means.** Instead of one big application (a **monolith**), you split functionality into several small, independently deployable services, each owning one capability (e.g. `auth`, `users`, `orders`, `payments`) and its own data, communicating over the network (HTTP/REST, gRPC, or a message queue). Each runs in its own container. The reverse proxy (Step 23) gives clients a single front door while many services run behind it.
- **The Linux/Docker foundations you already have** are exactly what microservices are built on:
  - **One service per container** (Step 21) — isolation, independent lifecycle.
  - **Service-name networking** (Step 24) — `orders` calls `users` at `http://users:8000`; Compose/orchestrator DNS resolves it. No hardcoded IPs.
  - **Env-based config** (Step 18) — each service is configured by environment variables (the 12-factor model): connection strings, ports, feature flags, secrets injected at deploy.
  - **Processes, ports, logs** (Steps 11/16/20) — each service is a process listening on a port, logging to stdout, supervised by the runtime's restart policy.
- **Internal vs edge.** Only the proxy is published to the outside; **service-to-service traffic stays on the internal network**. You can further segment with multiple Docker networks (e.g. a `frontend` network for proxy↔public-facing services and a `backend` network for services↔databases) so, say, the database is only reachable by the services that need it — defense in depth (Step 16's topology-as-firewall, taken further).
- **Health checks & resilience.** With many services, failures are normal, so each service exposes a **health endpoint** (`/healthz`) and a Docker `healthcheck`; the proxy/orchestrator routes only to healthy instances and restarts unhealthy ones. Services must tolerate a dependency being briefly down — **retries with backoff**, sensible timeouts, and not crash-looping. (`restart: unless-stopped` ≈ systemd's `Restart=on-failure` from Step 14.)
- **Scaling & load balancing.** Because a service is stateless and replaceable, you run **N copies** and load-balance across them: `docker compose up --scale orders=3`, with nginx `upstream` or Docker DNS round-robin spreading requests (Step 23). Stateless services scale horizontally; **state** (databases, sessions) is concentrated in a few stateful services with persistent volumes, or pushed to managed stores. "Keep services stateless; isolate state" is the core scaling rule.
- **Service communication patterns (overview):**
  - **Synchronous** — service A HTTP-calls service B and waits. Simple; couples their availability/latency.
  - **Asynchronous** — A publishes a message to a queue/broker (e.g. RabbitMQ, Redis, Kafka) and B consumes it later. Decouples services and absorbs load spikes, at the cost of complexity and eventual consistency. (Conceptual here; you'd add a broker as another Compose service.)
- **Observability becomes essential.** With one box you `tail` one log; with ten services you need **aggregated logs** (everything to stdout → a collector), **request tracing** (a request ID propagated across services so you can follow one call through the system), and **metrics/health dashboards**. Start simply: structured logs to stdout + `docker compose logs`, and grow from there.
- **Compose vs orchestrators.** Compose is perfect for development and small single-host deployments. At real scale you move to an orchestrator (**Kubernetes**, Nomad, ECS) that schedules containers across many machines, does rolling deploys, self-healing, and service discovery cluster-wide. The *concepts* are identical to what you're practicing — Kubernetes is "Compose's ideas, distributed." Learn it after you're fluent here.
- **The honest trade-off.** Microservices add real cost: network calls instead of function calls, distributed-systems failure modes, more moving parts to deploy/observe, and harder debugging. They pay off for large teams and independently scaling components — **not** for a small app, where a well-structured monolith on one server (everything you built in Steps 22–24) is faster to build and operate. Know *why* you're splitting before you split.

## Exercises
1. Map a small system: design 3 services behind your proxy — `web` (UI), `api` (business logic), `auth` (login) — plus a `db`. Draw who talks to whom, which are published vs internal, and which networks they sit on. Review with Claude.
2. Extend your Step 24 stack: add a second backend service (`auth`) and route `/auth/` to it from the proxy while `/api/` still goes to `api`. Confirm both reach their correct container via service name.
3. Service-to-service call: make `api` call `auth` internally (e.g. `api` fetches `http://auth:8000/whoami` on a request). From inside the `api` container, `curl http://auth:8000/...` to prove internal DNS works and that `auth` has **no** published port yet is reachable internally.
4. Add health endpoints + Docker healthchecks to each service; `docker compose ps` should show `healthy`. Then stop `auth` and watch `api`'s call fail — implement a short retry/backoff (Step 19 `until`/loop logic) and observe graceful behavior vs a hard crash.
5. Network segmentation: split into `frontend` and `backend` Docker networks so the `db` is only on `backend` (reachable by `api`/`auth`, not by `proxy`). Verify from inside `proxy` that `getent hosts db` fails, but from `api` it resolves.
6. Scale a stateless service: `docker compose up -d --scale api=3`, hit `/api/` repeatedly, and confirm requests spread across instances (log the hostname per instance to see it). Discuss with Claude why you can't naively `--scale db=3`.
7. Reflection: write 4–5 sentences on when you'd choose this microservice setup vs a single server running all services (Step 22–23 style). Be specific about team size, scaling needs, and operational cost. Ask Claude to challenge your reasoning.

## Best Practices & Pitfalls
- **Keep services stateless; concentrate state.** Stateless services scale by adding copies; anything holding state (DB, cache, sessions) needs persistent volumes and can't be casually replicated. Mixing state into every service is the fastest way to an unscalable mess.
- **Communicate by service name on an internal network; publish only the edge.** No hardcoded IPs, no exposing internal services to the host. Segment networks so each service reaches only what it needs.
- **Design for failure: health checks, timeouts, retries with backoff.** In a distributed system a dependency *will* be momentarily down. Services that assume 100% availability crash-loop and cascade. Fail gracefully and let the runtime restart them.
- **Log to stdout and aggregate; propagate a request ID.** Per-container log files don't scale. Structured stdout logs + a request ID threaded through calls is what makes debugging across services tractable.
- **Pitfall: premature microservices.** Splitting a small app into services multiplies operational burden (networking, deploys, observability, distributed bugs) for little benefit. Start with a clean monolith on one server; extract services when a real scaling/team boundary demands it.
- **Pitfall: a shared database across all services.** If every service reads/writes one DB, you've built a distributed monolith with all the cost and none of the independence. Give services their own data ownership (or be deliberate about the coupling).
- **Pitfall: ignoring the network's reality.** In-process calls are nanoseconds and never "fail"; network calls are milliseconds and fail routinely (timeouts, partial responses, retries causing duplicates). Treat every cross-service call as fallible.

## Checklist
- [ ] I can explain microservices vs a monolith and the Linux/Docker primitives they rely on.
- [ ] I can wire services together by service name on an internal network and publish only the edge.
- [ ] I can add health endpoints/healthchecks and handle a dependency being down (retry/backoff).
- [ ] I can segment networks and scale a stateless service while isolating stateful ones.
- [ ] I can articulate the trade-offs and say when a single server beats microservices.

## Resources
- 12-factor app (the microservice config/stateless bible): https://12factor.net/
- Microservices, the good and bad (Fowler): https://martinfowler.com/articles/microservices.html
- Compose multiple networks: https://docs.docker.com/compose/how-tos/networking/#specify-custom-networks
- Why containers don't run systemd / one-process model: https://docs.docker.com/engine/containers/multi-service_container/
