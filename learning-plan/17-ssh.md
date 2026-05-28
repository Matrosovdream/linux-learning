# 17 — SSH & Remote Access

## Goals
- Understand how SSH gives you a secure shell on a remote machine.
- Authenticate with key pairs (not passwords) and manage your keys.
- Copy files with `scp`/`rsync` and streamline connections with `~/.ssh/config`.

## Concepts
- **What SSH is.** **SSH (Secure Shell)** is an encrypted protocol for logging into and running commands on a remote machine over the network — the standard way you reach any Linux server, cloud VM, or `git push` over SSH. Default port **22**.
  - **Client:** `ssh user@host` opens a remote shell. `ssh user@host "command"` runs one command and returns.
  - **Server:** the **`sshd`** daemon listens on the remote box (configured in `/etc/ssh/sshd_config`). It's what you connect *to*.
- **Key-based authentication (how real servers work — far better than passwords):**
  - You generate a **key pair**: a **private key** (`~/.ssh/id_ed25519`, secret, stays on your machine, never shared) and a **public key** (`~/.ssh/id_ed25519.pub`, safe to hand out).
  - You put the **public** key in the server's `~/.ssh/authorized_keys`. SSH then proves you hold the matching private key — no password crosses the wire.
  - Generate: `ssh-keygen -t ed25519 -C "you@example"` (ed25519 is the modern default; RSA-4096 if you need compatibility). Optionally protect the private key with a passphrase.
  - Install your key on a server easily: `ssh-copy-id user@host` (appends your pubkey to the server's `authorized_keys`).
- **Permissions matter (SSH is strict on purpose):** `~/.ssh` must be `700`, the private key `600`, `authorized_keys` `600`. SSH **refuses** keys with looser permissions ("UNPROTECTED PRIVATE KEY FILE!"). This is the permissions lesson (Step 07) made real.
- **`~/.ssh/config` — stop typing long commands.** Define hosts once:
  ```
  Host myserver
      HostName 203.0.113.10
      User deploy
      Port 22
      IdentityFile ~/.ssh/id_ed25519
  ```
  Then just `ssh myserver`. Also powers `scp myserver:...`, git, etc.
- **Copying files over SSH:**
  - `scp file user@host:/path/` — copy a file up; `scp user@host:/path/file .` — copy down; `scp -r dir user@host:/path/` — recursive.
  - **`rsync`** — smarter copying: only transfers differences, can resume, preserves permissions. `rsync -avz ./site/ user@host:/var/www/html/` is the standard "deploy a directory" command (`-a` archive, `-v` verbose, `-z` compress; the trailing-slash semantics matter — `src/` copies contents, `src` copies the dir). Great for deployments.
- **Other power tools (know they exist):**
  - **Port forwarding/tunnels:** `ssh -L 5432:localhost:5432 user@host` securely tunnels a remote port to your machine — e.g. reach a database that only listens on the server's loopback. Hugely useful and secure.
  - **`ssh-agent`** caches your decrypted private key so you enter the passphrase once per session (`ssh-add`).
- **Hardening an SSH server** (what you'd set in `/etc/ssh/sshd_config` on a real box): `PasswordAuthentication no` (keys only), `PermitRootLogin no` (no direct root login — use a sudo user), a non-default port if you like, and `fail2ban` to throttle brute-force attempts. Reload with `systemctl reload ssh` after editing.
- **The container angle.** You already get a shell via `docker compose exec` — you don't *need* SSH to use the lab. But you can install and run `sshd` inside the container to practice the full client/server flow locally (generate keys, `ssh-copy-id`, log in, `scp`), which is exactly what you'll do against a cloud VM. The practice section walks through this.

## Exercises
1. Generate a key pair (inside the lab is fine for practice): `ssh-keygen -t ed25519 -C "lab key"` — accept the default path, optionally set a passphrase. Inspect: `ls -l ~/.ssh` and `cat ~/.ssh/id_ed25519.pub`. Note the strict permissions on the private key.
2. Run an SSH server in the lab to connect to: `apt install -y openssh-server && service ssh start` (or `/usr/sbin/sshd`). Confirm it's listening: `ss -tlnp | grep :22`.
3. Make a target user and install your key: `useradd -m -s /bin/bash deploy && passwd deploy`, then `ssh-copy-id deploy@localhost` (or manually append your pubkey to `/home/deploy/.ssh/authorized_keys` with `700`/`600` perms).
4. Connect by key: `ssh deploy@localhost` — you should log in *without* a password (key auth). Run `whoami`, then `exit`. If it asks for a password, debug permissions with Claude (`~/.ssh` 700, key 600).
5. Run a one-shot remote command: `ssh deploy@localhost "hostname && uptime"` — note it runs and returns without an interactive session.
6. `~/.ssh/config`: add a `Host lab` block pointing at `localhost` as user `deploy`, then connect with just `ssh lab`. Confirm it works.
7. Copy files: `scp /etc/hostname deploy@localhost:/tmp/`, then `ssh deploy@localhost "cat /tmp/hostname"`. Then try a directory sync: `rsync -av /work/lab/ deploy@localhost:/tmp/labcopy/` and verify on the other side.
8. Inspect server config: `grep -E "PasswordAuthentication|PermitRootLogin" /etc/ssh/sshd_config`. Discuss with Claude which values you'd set for a hardened public server.

## Best Practices & Pitfalls
- **Use keys, disable passwords on real servers.** Password auth invites brute-force attacks; key auth is both more secure and more convenient. Set `PasswordAuthentication no` once your key works.
- **Never share or commit your private key.** Only the `.pub` is shared. A leaked private key = full access. Keep private keys `600`, protect them with a passphrase, and use `ssh-agent` so you type it once.
- **Get the permissions right or SSH silently refuses keys.** `~/.ssh` = `700`, private key = `600`, `authorized_keys` = `600`. "Permission denied (publickey)" is very often a permissions/ownership problem, not a wrong key.
- **Disable direct root login (`PermitRootLogin no`); log in as a sudo user.** Auditable, and root over SSH is a prime target.
- **Pitfall: editing `sshd_config` and locking yourself out.** On a remote box, keep your current session open and test a *new* connection before closing the old one. A bad config + `reload` can otherwise strand you.
- **Pitfall: `rsync` trailing slashes.** `rsync -a src/ dst/` copies the *contents* of `src`; `rsync -a src dst/` copies `src` *itself* into `dst`. Getting this wrong nests or scatters files — test with `--dry-run` first.
- **Pitfall: host key warnings.** "REMOTE HOST IDENTIFICATION HAS CHANGED" means the server's key differs from what you cached in `~/.ssh/known_hosts` — expected after rebuilding a box, but on a real server consider whether it's a man-in-the-middle before clearing the entry.

## Checklist
- [ ] I can explain SSH (client/server, port 22) and why key auth beats passwords.
- [ ] I can generate a key pair and install the public key on a server (`ssh-copy-id`).
- [ ] I can log in by key, run one-shot remote commands, and use `~/.ssh/config` aliases.
- [ ] I can copy files with `scp` and sync directories with `rsync`.
- [ ] I know the correct `~/.ssh` permissions and the basics of hardening `sshd_config`.

## Resources
- SSH essentials (DigitalOcean): https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys
- `ssh-keygen`/`ssh` manuals: https://man.openbsd.org/ssh
- `rsync` how-to: https://www.digitalocean.com/community/tutorials/how-to-use-rsync-to-sync-local-and-remote-directories
- SSH config file reference: https://www.ssh.com/academy/ssh/config
