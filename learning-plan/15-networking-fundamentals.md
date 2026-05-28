# 15 — Networking Fundamentals

## Goals
- Build a mental model of IPs, ports, and DNS — the three things web traffic depends on.
- Inspect a host's network configuration with `ip`.
- Test connectivity and make HTTP requests from the command line with `ping`, `curl`, and `wget`.

## Concepts
- **The 30-second model of a web request.** Your browser wants `http://example.com/page`. It (1) resolves the **name** `example.com` to an **IP address** via **DNS**, (2) opens a **TCP** connection to that IP on a **port** (80 for HTTP, 443 for HTTPS), and (3) sends an HTTP request and gets a response. Every networking tool below pokes at one of those steps.
- **IP addresses** identify a machine on a network. **IPv4** looks like `192.168.1.10`; **IPv6** like `2001:db8::1`. Special ones you'll meet constantly:
  - `127.0.0.1` (a.k.a. **`localhost`**) — *this machine itself* (the loopback interface). A service bound here is reachable only locally.
  - `0.0.0.0` — "all interfaces" — a server bound here accepts connections from anywhere. (Binding `127.0.0.1` vs `0.0.0.0` is a frequent web-dev gotcha — see pitfalls.)
  - **Private ranges** (`10.x`, `172.16–31.x`, `192.168.x`) — internal networks, not routable on the public internet. Docker gives containers private IPs in such a range.
- **Ports** distinguish services on the same IP. A port is a number 0–65535. Well-known: **80** HTTP, **443** HTTPS, **22** SSH, **5432** Postgres, **3306** MySQL, **6379** Redis. Your dev apps often use **3000/8000/8080**. An address+port like `127.0.0.1:8080` is a **socket**. (Deep dive in Step 16.)
- **DNS** maps names to IPs. Tools: `getent hosts example.com` (the system resolver — respects `/etc/hosts`), `dig example.com` / `host example.com` / `nslookup example.com` (query DNS directly; `apt install dnsutils`). **`/etc/hosts`** is a local override file (`127.0.0.1 myapp.local`) checked before DNS — invaluable for testing virtual hosts locally. **`/etc/resolv.conf`** lists which DNS servers the box uses.
- **Inspecting your network with `ip`** (the modern tool; `ifconfig`/`netstat` from `net-tools` are legacy but still common in docs):
  - `ip addr` (or `ip a`) — interfaces and their IP addresses. Look for `lo` (loopback) and `eth0` (your main interface).
  - `ip route` (or `ip r`) — the routing table; the **default route** is your gateway to everything else.
  - `ip link` — network interfaces and their state (up/down).
  - `hostname -I` — quick list of this host's IPs.
- **Testing connectivity & making requests:**
  - `ping host` — is it reachable at the IP layer? (Sends ICMP echoes. `Ctrl+C` to stop. Note: many cloud hosts/containers block ICMP, so "no ping" ≠ "down".)
  - **`curl`** — the web dev's Swiss army knife. `curl http://localhost` (fetch a page), `curl -I url` (headers only — see status code), `curl -v url` (verbose: DNS, connection, request, response — *the* debugging mode), `curl -L` (follow redirects), `curl -o file url` (save), `curl -X POST -d 'data' url` (send data). You'll use `curl` to test every web service you build.
  - **`wget`** — better for downloading files: `wget url`, `wget -O name url`. `curl` for talking to APIs, `wget` for grabbing files (rough rule of thumb).
- **Web relevance:** "is my app reachable?", "what status/headers does it return?", "does this name resolve?", "is the service bound to the right interface?" — these are 90% of local web debugging, and they're all the commands above.

## Exercises
1. Install tools: `apt install -y iproute2 dnsutils curl wget net-tools iputils-ping`.
2. Map your container's network: `ip addr` (find `lo` = `127.0.0.1` and `eth0` = the container's private IP), `ip route` (note the default gateway — that's how it reaches out), `hostname -I`.
3. DNS: `getent hosts localhost`, then look at `cat /etc/hosts` and `cat /etc/resolv.conf`. If you have internet, `dig example.com +short` and compare to `getent hosts example.com`.
4. Local hostname override: add a line `echo "127.0.0.1 myapp.local" >> /etc/hosts`, then `getent hosts myapp.local` — you just invented a hostname. (You'll use this to test name-based routing in Part 7.)
5. `curl` basics against a public site (if online): `curl -I https://example.com` (read the status line + headers), then `curl -v https://example.com 2>&1 | head -40` and identify the DNS lookup, TCP connect, TLS handshake, and request lines.
6. Loopback vs all-interfaces demo: run a throwaway server — `python3 -m http.server 8000 --bind 127.0.0.1 &` — then `curl http://127.0.0.1:8000` (works). Note that bound to `127.0.0.1` it's local-only; rerun with `--bind 0.0.0.0` and discuss with Claude what changes for other containers. Kill it when done.
7. Save a download: `wget -O /work/lab/example.html https://example.com` (if online), then `head /work/lab/example.html`.

## Best Practices & Pitfalls
- **`127.0.0.1` vs `0.0.0.0` is the #1 "why can't I reach my app" bug.** A service bound to `127.0.0.1` is reachable only from *inside* the same machine/container. To accept connections from other containers or your host, bind `0.0.0.0`. (But don't expose internal services to `0.0.0.0` on a public box without a firewall — Step 16.)
- **`curl -v` is your first debugging move for any HTTP problem.** It shows exactly where things fail: name resolution, TCP connect, TLS, or the HTTP response. Learn to read its output.
- **"Ping fails" doesn't mean "server down."** ICMP is frequently blocked. Test the actual service with `curl` to its port instead of relying on `ping`.
- **`/etc/hosts` beats DNS — use it for local testing, but clean it up.** Stale entries cause baffling "it resolves to the wrong place" bugs months later.
- **Pitfall: confusing the host's IP with a container's IP.** In Docker, each container has its own private IP; `localhost` inside a container is the *container*, not your Mac. This trips up everyone wiring services together (clarified in Part 7).
- **Pitfall: using `ifconfig`/`netstat` and finding them missing.** They're legacy (`net-tools`); the modern equivalents are `ip` and `ss`. Know both since old tutorials use the former.

## Checklist
- [ ] I can sketch the DNS → IP → port → HTTP request flow.
- [ ] I can read `ip addr` and `ip route` and identify loopback, my interface, and the gateway.
- [ ] I can resolve a name (`getent`/`dig`) and add a local override in `/etc/hosts`.
- [ ] I can fetch a URL and inspect headers/verbose output with `curl -I` / `curl -v`.
- [ ] I understand the difference between binding `127.0.0.1` and `0.0.0.0`.

## Resources
- `ip` command primer: https://www.redhat.com/sysadmin/ip-command
- `curl` cookbook: https://everything.curl.dev/
- How DNS works (illustrated): https://howdns.works/
- TCP/IP basics: https://www.cloudflare.com/learning/ddos/glossary/tcp-ip/
