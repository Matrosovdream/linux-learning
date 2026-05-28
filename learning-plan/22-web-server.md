# 22 — Running a Web Server

## Goals
- Install and run **nginx** on your Linux server and serve a static site.
- Understand nginx's config layout (`nginx.conf`, `sites-available`/`sites-enabled`, `server`/`location` blocks).
- Read access and error logs to debug what the server is actually doing.

## Concepts
- **What a web server does.** It listens on a TCP port (80 for HTTP, 443 for HTTPS), accepts requests, and returns responses — either **static files** from disk or, by proxying, responses from an **application** behind it (Step 23). **nginx** is the most common choice for web devs: fast, simple config, great as both a static server and a reverse proxy.
- **Installing & controlling nginx** (Debian): `apt install -y nginx`. Control it via systemd where available (`systemctl start/reload nginx`) or directly: `nginx` (start), `nginx -s reload` (reload config, no downtime), `nginx -s stop`, and crucially **`nginx -t`** (test config syntax — *always* run before reloading).
- **The config layout (Debian convention):**
  - `/etc/nginx/nginx.conf` — the **main** config: global settings, then `http { ... }` which `include`s the site configs.
  - `/etc/nginx/sites-available/` — one file **per site** (whether or not it's active). You author here.
  - `/etc/nginx/sites-enabled/` — **symlinks** to the files in `sites-available` that are actually live. **Enable** a site by symlinking it (`ln -s ../sites-available/mysite .`), **disable** by removing the symlink — the config file stays. (This is exactly the symlink pattern from Step 05, and `conf.d/` is an alternative drop-in dir.)
  - `/var/www/html/` — the default document root (where the files served live). `/var/log/nginx/` — `access.log` and `error.log`.
- **`server` and `location` blocks (the core of nginx config):**
  ```nginx
  server {
      listen 80;
      server_name myapp.local;          # which hostname this block answers for
      root /var/www/myapp;              # where files are served from
      index index.html;

      location / {                       # match request paths starting with /
          try_files $uri $uri/ =404;     # serve the file, or a dir, else 404
      }

      location /assets/ {                # a more specific path gets its own rules
          expires 7d;                    # cache static assets in the browser
      }
  }
  ```
  - A **`server`** block is a virtual host: nginx picks one by the request's `Host` header matching `server_name` (this is **name-based virtual hosting** — many sites on one IP/port). Listening on the same port with different `server_name`s is how one server hosts several sites.
  - A **`location`** block matches request paths and decides how to handle them (serve files, proxy, redirect, set headers). More-specific locations win.
- **The request → file mapping.** For `GET /about.html` with `root /var/www/myapp`, nginx serves `/var/www/myapp/about.html`. `try_files` controls the fallback chain. Getting `root` + permissions right (the `www-data` user must be able to read the files and traverse the dirs — Step 07!) is where most "403 Forbidden"/"404"/"file not found" issues come from.
- **Reading the logs (your feedback loop):** `tail -f /var/log/nginx/access.log` shows each request with its status code; `tail -f /var/log/nginx/error.log` shows *why* something failed (permission denied, file not found, upstream error). Keep both open in split terminals while you test with `curl`. The access log format ties straight back to the `awk`/`grep` analysis from Steps 09/20.
- **HTTP status codes you'll see constantly:** `200` OK, `301/302` redirect, `403` forbidden (usually permissions), `404` not found (wrong path/root), `500` server error, `502` bad gateway / `504` timeout (the upstream app is down/slow — Step 23). Learning to map a status code to a cause is core web-server debugging.
- **Where this fits the growing server.** This is the first *service* on your server. You'll install nginx, serve a static page, then in Step 23 turn it into a reverse proxy in front of app services, and in Steps 24–26 wire a whole stack behind it.

## Exercises
1. Install and verify: `apt install -y nginx`, start it (`nginx` or `service nginx start`), then `ss -tlnp | grep nginx` (listening on 80) and `curl -I http://127.0.0.1` (expect `200`). View the default page: `curl http://127.0.0.1 | head`.
2. Tour the config: `ls /etc/nginx/`, `ls -l /etc/nginx/sites-enabled/` (note the symlink → `sites-available/default`), and read `/etc/nginx/sites-available/default`. Map every directive to the concepts above with Claude.
3. Serve your own static site: create `/var/www/myapp/index.html` with some HTML and an `/var/www/myapp/about.html`. Set ownership/permissions so `www-data` can read it (`chown -R www-data:www-data /var/www/myapp`, dirs `755`, files `644`).
4. Write a site config: create `/etc/nginx/sites-available/myapp` with a `server` block (`listen 80; server_name myapp.local; root /var/www/myapp; index index.html;` and a `location /` with `try_files`). Enable it: `ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/`.
5. Test before reloading: `nginx -t` (fix any syntax error it reports), then `nginx -s reload`. Add `127.0.0.1 myapp.local` to `/etc/hosts` (Step 15), then `curl http://myapp.local/` and `curl http://myapp.local/about.html`.
6. Break it on purpose, read the logs: `chmod 000 /var/www/myapp/index.html`, `curl -I http://myapp.local/` (now `403`), and `tail -n 5 /var/log/nginx/error.log` to see the "Permission denied". Fix perms, confirm `200`. Then request a missing file for a `404` and watch both logs.
7. Disable cleanly: `rm /etc/nginx/sites-enabled/myapp`, `nginx -t && nginx -s reload`, confirm the site is gone but `/etc/nginx/sites-available/myapp` remains. Re-enable it.

## Best Practices & Pitfalls
- **Always `nginx -t` before reloading.** A syntax error on `reload` can take the server down or refuse to start. Test, *then* `nginx -s reload`. Make it a reflex.
- **`reload`, don't `restart`.** `nginx -s reload` (or `systemctl reload`) applies config with zero dropped connections; a restart briefly drops them. (Same lesson as Steps 11/14.)
- **403/404 are almost always `root` + permissions.** Check that `root` points where your files actually are, that `www-data` can read the files (`644`) and traverse *every* parent dir (`755`, the directory-`x` rule from Step 07), and read `error.log` — it tells you exactly which path failed.
- **Enable sites by symlink, keep the source in `sites-available`.** This lets you disable a site without deleting its config, and keeps your configs reviewable in one place. (It's the Step 05 symlink pattern doing real work.)
- **Pitfall: editing config and forgetting to reload.** Your change does nothing until `nginx -s reload`. If "the change isn't taking," you probably edited the file but didn't reload (or edited the wrong file — check the symlink target).
- **Pitfall: matching the wrong `server` block.** If `server_name` doesn't match the request's `Host`, nginx falls back to the default server and you get the wrong site. Use `curl -H "Host: myapp.local" ...` or `/etc/hosts` to test name-based hosts correctly.
- **Pitfall: `index.html` ownership.** Files created as root in `/var/www` may be unreadable/owned wrong for `www-data`. `chown` them to the web user (or ensure world-read) — don't `chmod 777`.

## Checklist
- [ ] I can install nginx, start it, and confirm it's serving with `ss` + `curl -I`.
- [ ] I can explain `nginx.conf`, `sites-available`/`sites-enabled` (symlinks), and `server`/`location` blocks.
- [ ] I can serve a custom static site with correct `root` and `www-data` permissions.
- [ ] I always run `nginx -t` before `nginx -s reload`.
- [ ] I can read access/error logs and map status codes (403/404/502) to causes.

## Resources
- nginx beginner's guide: https://nginx.org/en/docs/beginners_guide.html
- Debian nginx layout & directory structure: https://www.digitalocean.com/community/tutorials/understanding-the-nginx-configuration-file-structure-and-configuration-contexts
- nginx `server`/`location` matching: https://nginx.org/en/docs/http/request_processing.html
- HTTP status codes: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status
