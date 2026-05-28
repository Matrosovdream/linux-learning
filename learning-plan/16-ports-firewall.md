# 16 — Ports, Sockets & Firewall

## Goals
- See exactly which ports are open and which process owns each one.
- Understand listening vs established sockets and how traffic flows in.
- Control access with a firewall (`ufw`), and understand `iptables`/`nftables` underneath.

## Concepts
- **A socket = IP + port + protocol.** A server **listens** on a socket (e.g. `0.0.0.0:80` for a public web server); a client connects from an ephemeral high port. Knowing what's listening — and on which interface — is fundamental to running and securing services.
- **`ss` — the modern tool to inspect sockets** (replaces `netstat`):
  - `ss -tlnp` — the command you'll run most: **t**cp, **l**istening, **n**umeric (don't resolve names — faster), **p**rocess (show which program owns the socket). Output shows e.g. `LISTEN 0 511 0.0.0.0:80 ... users:(("nginx",pid=…))`.
  - `ss -tlnp | grep :80` — "is anything listening on 80, and what?"
  - `ss -tunap` — add **u**dp and **a**ll (established + listening). `ss -s` — summary stats.
  - Legacy equivalent: `netstat -tlnp` (from `net-tools`) — same idea, common in old docs.
- **Reading the address column:** `0.0.0.0:80` = listening on *all* interfaces (reachable externally); `127.0.0.1:5432` = listening only on loopback (local-only — a good default for databases); `[::]:80` = the IPv6 equivalent of all-interfaces.
- **Finding "what's using this port?"** When a service won't start because "address already in use": `ss -tlnp | grep :8080` shows the offending PID, then `kill` it (Step 11) or change your port. `lsof -i :8080` (install `lsof`) is another way.
- **Firewalls — controlling who can reach which port:**
  - A firewall filters packets by port/protocol/source. The kernel mechanism is **netfilter**, configured historically via **`iptables`** and now **`nftables`**. These are powerful but verbose.
  - **`ufw` (Uncomplicated Firewall)** is the friendly front-end on Debian/Ubuntu and what you should use day to day:
    - `ufw status verbose` — current rules. `ufw enable`/`disable` — turn it on/off.
    - `ufw allow 22/tcp` (SSH — allow this *before* enabling, or you can lock yourself out of a remote box!), `ufw allow 80,443/tcp` (web), `ufw deny 3306` (block MySQL from outside).
    - `ufw allow from 10.0.0.0/24 to any port 5432` — allow Postgres only from a specific subnet.
    - Default stance: `ufw default deny incoming` + `ufw default allow outgoing` — deny everything inbound, then open only the ports you need.
  - **`iptables -L -n`** / **`nft list ruleset`** show the low-level rules. You'll mostly read these to debug, and let `ufw` (or Docker) write them.
- **The container/Docker reality (important):**
  - In a single learning container you typically won't run `ufw` (containers usually have no firewall and no privilege to manage netfilter). The real "firewall" for containers is **which ports you publish** in `docker-compose.yml` (`ports: "8080:80"`) and **Docker networks** (Part 7). Unpublished ports aren't reachable from your host — that *is* your access control.
  - On a real VM hosting Docker, you'd use `ufw`/cloud security groups for the host *and* be careful that Docker can bypass `ufw` by writing its own iptables rules (a well-known gotcha). For now: learn `ss` thoroughly (works everywhere) and understand `ufw` conceptually for when you deploy to a VM.
- **Web relevance:** before you can route traffic (Part 7) you must answer "is my service actually listening, on which interface, and is the port reachable?" `ss -tlnp` answers the first two; published ports / firewall rules answer the third.

## Exercises
1. Install tools: `apt install -y iproute2 lsof net-tools`. (`ss` is in `iproute2`.)
2. Baseline: `ss -tlnp` on the fresh container — likely little or nothing is listening. Note the empty-ish state.
3. Create a listener and find it: `python3 -m http.server 8080 --bind 0.0.0.0 &`, then `ss -tlnp | grep :8080` — confirm it's listening on `0.0.0.0:8080` and see the owning PID. `curl http://127.0.0.1:8080` to prove it serves.
4. Loopback vs all-interfaces: kill it, restart with `--bind 127.0.0.1`, and re-check `ss -tlnp` — note the address changed to `127.0.0.1:8080`. Discuss with Claude who can now reach it.
5. "Port in use" drill: with the server still running, start another on the same port — read the "Address already in use" error, use `ss -tlnp | grep :8080` to find the holder, kill it, then start successfully.
6. Install nginx and observe real listeners: `apt install -y nginx`, start it (`nginx` or `service nginx start`), then `ss -tlnp | grep nginx` — see ports 80 (and the master/worker processes). `curl -I http://127.0.0.1` for the default page.
7. Firewall (conceptual / if available): `apt install -y ufw`, then `ufw status`. Sketch (don't necessarily enable in the container) the rule set you'd use on a public web VM: deny incoming by default, allow 22/80/443. Ask Claude why allowing 22 *before* `ufw enable` matters.

## Best Practices & Pitfalls
- **`ss -tlnp` is the first thing to run when a service "isn't working."** It tells you instantly whether the service is even listening and on which interface — before you blame networking, DNS, or the firewall.
- **Bind databases/internal services to `127.0.0.1` (or a private network), never `0.0.0.0`, unless they must be public.** A Postgres listening on `0.0.0.0` with a weak password is how servers get owned. Default to local-only and expose deliberately.
- **On a remote box, allow SSH (22) in the firewall *before* enabling it.** `ufw enable` with no allow rule for 22 cuts your own connection — a classic, painful self-lockout. (In the container you have console access, so it's safe to experiment.)
- **Default-deny inbound, allow-list the rest.** Open only 22/80/443 (and only what's truly needed). Every open port is attack surface.
- **Pitfall: Docker + ufw.** Docker publishes ports by editing iptables directly and can bypass `ufw` rules, so a "blocked" port may still be reachable via a published container. On real hosts, control container exposure through `ports:`/Docker networks, not just `ufw`.
- **Pitfall: confusing "listening" with "reachable."** A service can listen fine yet be unreachable because the port isn't published (Docker), a firewall blocks it, or it's bound to the wrong interface. Check all three layers.

## Checklist
- [ ] I can list listening ports and their owning processes with `ss -tlnp`.
- [ ] I can read whether a socket is bound to `127.0.0.1`, `0.0.0.0`, or a specific IP.
- [ ] I can find and resolve an "address already in use" conflict.
- [ ] I understand `ufw`'s default-deny-inbound model and the SSH-before-enable rule.
- [ ] I understand that in Docker, published ports / networks are the real access control.

## Resources
- `ss` usage: https://www.redhat.com/sysadmin/ss-command
- `ufw` guide (Ubuntu): https://help.ubuntu.com/community/UFW
- Docker and iptables/ufw caveat: https://docs.docker.com/engine/network/packet-filtering-firewalls/
- `man ss`, `man ufw`, `man iptables`
