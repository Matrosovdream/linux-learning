# 23 — Reverse Proxy & Routing

## Goals
- Understand what a reverse proxy is and why nearly every web stack has one in front.
- Configure nginx to route requests to backend application services by path and by hostname.
- Understand upstreams, proxy headers, and the basics of adding TLS (HTTPS).

## Concepts
- **What a reverse proxy is.** A server that sits *in front of* your application(s), receives client requests, and forwards them to the right backend, then relays the response back. nginx is the canonical reverse proxy. Why every real stack uses one:
  - **Routing** — one public entry point (port 80/443) fans out to many backend services by URL path or hostname.
  - **TLS termination** — handle HTTPS in one place; backends speak plain HTTP internally.
  - **Cross-cutting concerns** — caching, gzip, rate limiting, request size limits, security headers, load balancing — all in one layer your app doesn't have to implement.
  - **Decoupling** — clients hit `:443`; backends can be any language on any internal port, restarted/scaled independently.
- **`proxy_pass` — the core directive.** Inside a `location`, forward matching requests to a backend:
  ```nginx
  location /api/ {
      proxy_pass http://127.0.0.1:3000;     # forward to the app on port 3000
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
  }
  ```
  - The **proxy headers** matter: without them the backend sees nginx's IP and loses the original host/client/scheme. Apps use `X-Forwarded-*` to know the real client and whether the original request was HTTPS. Forgetting these causes wrong redirects, broken logging, and "why does my app think every request comes from 127.0.0.1?" bugs.
- **Path-based routing** — split one hostname across services by URL prefix:
  - `location / { proxy_pass http://frontend; }` — the web UI.
  - `location /api/ { proxy_pass http://api; }` — the API service.
  - `location /auth/ { proxy_pass http://auth; }` — the auth service.
  - This is how a microservice stack presents a single origin to the browser. (Mind the trailing-slash semantics of `proxy_pass` — a `/` on the target rewrites the path prefix; without it, the full path is passed through. Test both.)
- **Host-based routing** — split by `Host` header using multiple `server` blocks (`server_name api.myapp.local;` vs `app.myapp.local;`), each proxying elsewhere. Combine with `/etc/hosts` entries to test locally (Step 15).
- **`upstream` blocks — naming and load-balancing backends:**
  ```nginx
  upstream api_backend {
      server 127.0.0.1:3000;
      server 127.0.0.1:3001;     # add more instances → nginx load-balances (round-robin by default)
  }
  server { location /api/ { proxy_pass http://api_backend; } }
  ```
  This is your on-ramp to scaling: run several copies of a service and let nginx spread load. In Docker (Step 24), `server` lines become container/service names resolved by Docker's internal DNS.
- **Health & failure modes.** When the backend is down or slow, the proxy returns **502 Bad Gateway** (can't connect) or **504 Gateway Timeout** (no response in time). These are *the* signal that "nginx is fine, the app behind it isn't" — distinct from a 404 (routing/path) or 403 (permissions). Reading which code you got tells you which layer to debug.
- **TLS / HTTPS basics.** In production you terminate TLS at nginx: `listen 443 ssl;` with `ssl_certificate`/`ssl_certificate_key` pointing at a cert. Certs come free from **Let's Encrypt** via **certbot** (which can auto-edit nginx config and renew). For local practice you generate a **self-signed** cert (browsers warn, but the mechanics are identical). Always redirect 80 → 443. Private key files must be `600` (Step 07).
- **Where this fits the growing server.** You already serve static files (Step 22). Now you put nginx *in front of* one or more small backend apps (a tiny Python/Node server, or another container), routing by path. In Steps 24–26 those backends become separate Compose services and nginx routes to them by service name — a real reverse-proxy-fronted microservice stack.

## Exercises
1. Stand up a backend app to proxy to: `python3 -m http.server 3000 --bind 127.0.0.1 &` (a stand-in "app" on port 3000). Confirm `curl http://127.0.0.1:3000` works locally.
2. Proxy to it: in your `myapp` site config add `location /api/ { proxy_pass http://127.0.0.1:3000/; ... }` with the four `proxy_set_header` lines. `nginx -t && nginx -s reload`, then `curl http://myapp.local/api/` — you reached the backend *through* nginx on port 80.
3. Trailing-slash experiment: try `proxy_pass http://127.0.0.1:3000;` (no trailing slash) vs `.../;` (with), reload between, and `curl http://myapp.local/api/something` each time. Observe how the upstream path differs. Explain the rule to Claude.
4. Two backends + path routing: start a second app on 3001, add `location /app2/ { proxy_pass http://127.0.0.1:3001/; }`. Confirm `/api/` and `/app2/` reach different backends through the one nginx port.
5. See a 502: kill the port-3000 app, `curl -I http://myapp.local/api/` → `502 Bad Gateway`, and check `/var/log/nginx/error.log` for the "connect() failed" line. Restart the app, confirm it recovers. Contrast with a 404 (wrong path) and a 403 (perms).
6. `upstream` + headers: define an `upstream api_backend { server 127.0.0.1:3000; }`, point `/api/` at it, reload, and verify it still works. Then add a second `server` line (3001) and discuss round-robin load balancing with Claude.
7. (Stretch) Self-signed TLS: generate a cert with `openssl req -x509 -newkey rsa:2048 -nodes -keyout /etc/ssl/private/myapp.key -out /etc/ssl/certs/myapp.crt -days 365 -subj "/CN=myapp.local"`, set `chmod 600` on the key, add a `listen 443 ssl;` server block, reload, and `curl -k https://myapp.local/`. Add an 80→443 redirect.

## Best Practices & Pitfalls
- **Always set the `X-Forwarded-*` and `Host` headers when proxying.** Otherwise the backend logs nginx's IP, builds wrong absolute URLs/redirects, and can't tell HTTP from HTTPS. These four headers are non-negotiable for a correct proxy.
- **Mind `proxy_pass` trailing-slash semantics.** `proxy_pass http://app/;` (with `/`) replaces the matched `location` prefix; without it, the URI is passed through unchanged. This single character causes a huge share of "my routes 404 behind the proxy" bugs — test explicitly.
- **Learn the 502 vs 504 vs 404 vs 403 distinction cold.** 502/504 = backend down/slow (debug the app), 404 = routing/path (debug nginx `location`), 403 = permissions (debug filesystem). The status code points you at the right layer instantly.
- **Terminate TLS at the proxy; keep backends on plain HTTP internally.** Don't make every microservice manage certs. One TLS config at the edge, private network behind it. Redirect 80→443 and keep key files `600`.
- **Pitfall: proxying to `localhost` in a containerized world.** Inside a container, `127.0.0.1` is *that container*, not another service. When backends become separate containers (Step 24), `proxy_pass` must target the **service name** (Docker DNS), e.g. `http://api:3000`, not `127.0.0.1`. This is the #1 reverse-proxy-in-Docker mistake.
- **Pitfall: forgetting `nginx -t`/reload after edits** (same as Step 22) — and editing the wrong `server` block so requests match a different one. Use explicit `Host` headers / hostnames when testing.

## Checklist
- [ ] I can explain what a reverse proxy does and why every stack has one.
- [ ] I can configure `proxy_pass` with the required `Host`/`X-Forwarded-*` headers.
- [ ] I can route by path (`location`) and by host (`server_name`) to different backends.
- [ ] I understand `upstream` blocks and basic round-robin load balancing.
- [ ] I can read 502/504 vs 404/403 to know which layer is broken, and I know the TLS-at-the-edge model.

## Resources
- nginx reverse proxy guide: https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/
- `proxy_pass` & trailing slash explained: https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass
- nginx load balancing: https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/
- Let's Encrypt / certbot: https://certbot.eff.org/
