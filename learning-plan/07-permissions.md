# 07 — Permissions & Ownership

## Goals
- Read the permission string in `ls -l` and know exactly what it allows.
- Change permissions (`chmod`) and ownership (`chown`) with confidence.
- Understand users, groups, and `sudo` — and why web servers run as limited users.

## Concepts
- **Every file has an owner, a group, and three permission sets.** `ls -l` shows it:
  ```
  -rw-r--r-- 1 root  www-data  1024 May 28 10:00 index.html
  ```
  - First char: type — `-` file, `d` directory, `l` symlink.
  - Next 9 chars: three triplets of `rwx` for **owner**, **group**, **others**.
  - `root` = owning user, `www-data` = owning group.
- **The three permissions:**
  - **`r` (read)** — view file contents / list a directory.
  - **`w` (write)** — modify a file / create-delete entries in a directory.
  - **`x` (execute)** — run a file as a program / **enter** (cd into) a directory.
  - For directories, `x` is what lets you traverse into them — a common gotcha (read without execute on a dir is nearly useless).
- **Three identities:** **owner** (the user who owns it), **group** (a named set of users), **others** (everyone else). Linux checks them in that order and applies the *first* matching set.
- **Numeric (octal) mode** — each triplet is a digit: `r=4, w=2, x=1`, summed.
  - `7 = rwx`, `6 = rw-`, `5 = r-x`, `4 = r--`, `0 = ---`.
  - `chmod 644 file` → owner `rw-`, group `r--`, others `r--` (typical for a static file).
  - `chmod 755 dir` → owner `rwx`, group/others `r-x` (typical for a directory or executable).
  - `chmod 600 secret` → owner `rw-`, nobody else (private keys, env files).
- **Symbolic mode** — relative changes: `chmod +x script.sh` (add execute for all), `chmod u+x` (owner only), `chmod g-w` (remove group write), `chmod o= file` (others get nothing). Often clearer for a single tweak.
- **Changing ownership:** `chown user file`, `chown user:group file`, `chown -R user:group dir` (recursive). Only root can give a file to another user.
- **Changing group only:** `chgrp group file` (or `chown :group file`).
- **Users, groups & root:**
  - **root** (UID 0) is the superuser — bypasses all permission checks. In your container you're root by default, which is why nothing is blocked yet. On real servers you log in as a normal user.
  - **`sudo`** runs a single command as root: `sudo apt update`. It's how normal users do admin tasks without logging in as root. (Full treatment in Step 12.)
  - **service users** like `www-data` (the web server user on Debian) own and run web processes. This is deliberate: if nginx is compromised, the attacker is `www-data`, not root.
- **Why this matters for web dev:**
  - "Permission denied" from nginx usually means the web files or their parent directories aren't readable/traversable by `www-data`.
  - Private keys and `.env` files must be `600` (owner-only) or tools like SSH will refuse to use them.
  - Upload directories need `www-data` write access — but *only* those, never the whole tree.
- **`umask`** — the default permissions newly created files get (subtracts from `666`/`777`). `umask` shows it; usually `022`, giving new files `644` and new dirs `755`.

## Exercises
1. Read permissions: `ls -l /etc/passwd /etc/shadow /bin/ls`. For each, say out loud who can read, write, execute. (`/etc/shadow` being unreadable to others is a security feature — ask Claude why.)
2. Make an executable script: `echo '#!/bin/bash' > /work/lab/hi.sh && echo 'echo hello' >> /work/lab/hi.sh`. Try to run it: `/work/lab/hi.sh` (fails — no `x`). Then `chmod +x /work/lab/hi.sh` and run it again.
3. Octal practice: `chmod 600 /work/lab/hi.sh; ls -l /work/lab/hi.sh`, then `chmod 755`, then `644`. Predict the `rwx` string each time *before* you run `ls`.
4. Symbolic practice: starting from `644`, run `chmod u+x,g-r /work/lab/hi.sh` and read the result. Reset with `chmod 644`.
5. Ownership: `useradd -m alice` (root can do this), then `chown alice:alice /work/lab/hi.sh` and confirm with `ls -l`. Then give it to the web group: `chown root:www-data /work/lab/hi.sh` (the `www-data` group may need nginx installed first — note the error if so).
6. Directory `x` gotcha: `mkdir -p /work/lab/locked && chmod 600 /work/lab/locked` then try `ls /work/lab/locked` and `cd /work/lab/locked`. Observe what removing `x` from a directory breaks. Restore with `chmod 755`.
7. Inspect `umask`: run `umask`, create a new file with `touch /work/lab/new.txt`, and check its permissions match what the umask predicts.

## Best Practices & Pitfalls
- **Least privilege, always.** Give the minimum needed: `644` for static files, `755` for dirs and executables, `600` for secrets. Never "fix" a permission problem with `chmod 777` — that makes a file world-writable and is a classic security hole.
- **`600` for secrets is mandatory, not optional.** SSH private keys, `.env` files, TLS keys must be owner-only. SSH literally refuses keys that are group/world-readable (you'll hit this in Step 17).
- **Web "permission denied" is usually a directory-traversal issue.** For `www-data` to read `/var/www/html/index.html`, *every* directory in the path needs `x` for others/group. Check the whole chain, not just the file.
- **Pitfall: recursive `chmod` on a mixed tree.** `chmod -R 644` strips `x` from directories and breaks traversal. Use `chmod -R` carefully, or apply different modes to files vs dirs (`find ... -type d -exec chmod 755` / `-type f -exec chmod 644`).
- **Pitfall: being root masks permission bugs.** In your container everything works because you're root. When you later run services as `www-data`, permissions suddenly matter — test as the real service user.
- **Pitfall: `chown` needs root.** A normal user can't give files away; you'll need `sudo chown` on real servers.

## Checklist
- [ ] I can read an `ls -l` permission string and say who can read/write/execute.
- [ ] I can convert between octal (`644`, `755`, `600`) and `rwx`.
- [ ] I can use both numeric and symbolic `chmod`, and `chown user:group`.
- [ ] I understand directory `x` = "can enter," and why removing it breaks access.
- [ ] I can explain why web servers run as `www-data` and why secrets must be `600`.

## Resources
- `man chmod`, `man chown`, `man umask`
- Permissions explained (Ubuntu docs): https://help.ubuntu.com/community/FilePermissions
- Octal permission calculator (to check yourself): https://chmod-calculator.com/
- Why `www-data`: https://wiki.debian.org/WWW
