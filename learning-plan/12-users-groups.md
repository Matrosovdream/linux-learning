# 12 — Users, Groups & sudo

## Goals
- Understand Linux's user/group model and where it's stored.
- Create and manage users and groups.
- Use `sudo` correctly, and understand service accounts like `www-data`.

## Concepts
- **Users and UIDs.** Every account has a numeric **UID**. `root` is **UID 0** (the superuser — bypasses all permission checks). Convention on Debian/Ubuntu: UIDs `1–999` are **system/service accounts** (like `www-data`, `sshd`), `1000+` are **human users**.
- **Where users are defined (plain text files):**
  - `/etc/passwd` — one line per user: `name:x:UID:GID:comment:home:shell`. The `x` means "password is in `/etc/shadow`." Readable by all.
  - `/etc/shadow` — hashed passwords + expiry. Readable only by root (a security boundary — try `cat` as non-root and you'll be denied).
  - `/etc/group` — group definitions and membership: `groupname:x:GID:members`.
  - You *can* read these with `cat`/`grep`, but edit them with the proper tools (below), not by hand.
- **The login shell** (last field of `/etc/passwd`): human users get `/bin/bash`; service accounts often get `/usr/sbin/nologin` or `/bin/false` so nobody can log in *as* them — they exist only to own processes/files. That's a deliberate security design.
- **Managing users (need root):**
  - `useradd -m -s /bin/bash alice` — create `alice` with a home dir (`-m`) and bash shell. (`adduser alice` is Debian's friendlier interactive wrapper.)
  - `passwd alice` — set/change a password. `usermod` — modify (e.g. `usermod -aG sudo alice` adds alice to the `sudo` group; `-aG` = **append** to groups, the `a` is critical).
  - `userdel -r alice` — delete user and home (`-r`). `id alice` — show a user's UID, GID, and groups. `groups alice` — just the groups.
- **Managing groups:** `groupadd devs`, `gpasswd -a alice devs` (add member), `gpasswd -d alice devs` (remove). Groups are how you grant several users shared access to files (recall the group permission triplet from Step 07).
- **Switching identity:**
  - `su - alice` — start a full login shell as alice (needs alice's password; the `-` loads her environment). `su -` alone → root. `exit` returns.
  - `whoami` shows who you currently are; `id` shows the full picture.
- **`sudo` — run one command as root (the right way to do admin):**
  - `sudo apt update` — runs that command as root, asks for *your* password, logs it. Far safer than living as root.
  - Membership in the **`sudo`** group (Debian/Ubuntu) grants this. Rules live in `/etc/sudoers` — edit it **only** with `visudo` (it validates syntax; a broken sudoers file can lock you out).
  - `sudo -i` / `sudo su -` opens a root shell (use sparingly). `sudo -u www-data cmd` runs as a *specific* user — exactly how you test "can the web user read this file?"
- **Why your container feels different:** you're root by default in the lab, so nothing prompts for a password and everything is permitted. On a real server you operate as a normal user and reach for `sudo`. To practice realistically, create `alice`, give her sudo, and `su - alice`.
- **Service accounts & web dev:** nginx workers run as **`www-data`**; databases run as `postgres`/`mysql`. These users own only what they need. When you hit "permission denied" serving files, the fix is usually aligning file ownership/permissions with the service's user — and `sudo -u www-data` lets you reproduce the failure.

## Exercises
1. Inspect yourself and the system: `whoami`, `id`, then `cat /etc/passwd | column -t -s:` (readable table). Find `root` (UID 0) and any service accounts; note their shells (`nologin`/`false`).
2. Prove the shadow boundary: `cat /etc/shadow | head` as root (works), then create a normal user and try the same as them (denied). Ask Claude why this separation exists.
3. Create a human user: `useradd -m -s /bin/bash alice && passwd alice`, then `id alice` and `ls -ld /home/alice`.
4. Switch users: `su - alice`, run `whoami`/`pwd` (you're in `/home/alice`), create a file, then `exit` back to root. Notice alice can't read root's files.
5. Grant sudo: `usermod -aG sudo alice`, then `su - alice` and run `sudo cat /etc/shadow | head` (now permitted via sudo). Compare to step 2.
6. Groups for shared access: `groupadd devs && gpasswd -a alice devs && id alice`. Then make a shared dir: `mkdir /work/shared && chgrp devs /work/shared && chmod 770 /work/shared` and reason about who can write.
7. Act as a service user: `sudo -u nobody whoami` and `sudo -u nobody cat /etc/shadow` (denied — `nobody` has no rights). This is the pattern for testing `www-data` access later.

## Best Practices & Pitfalls
- **Don't live as root.** Use a normal user + `sudo` for admin tasks. It limits blast radius (a typo as root can wreck the system), creates an audit trail, and forces a deliberate pause before privileged actions. The container defaults to root for convenience — simulate real life with `alice`.
- **`usermod -aG` — never forget the `a`.** `usermod -G devs alice` *replaces* alice's entire group list with just `devs`, silently removing her from `sudo` and others. `-aG` **appends**. This mistake locks people out of sudo constantly.
- **Edit `/etc/sudoers` only with `visudo`.** It syntax-checks before saving. A malformed sudoers file can leave you unable to use sudo at all — painful on a remote box.
- **Group changes take effect on next login.** After adding a user to a group, they must log out/in (or `su - user` again) for it to apply — the current session keeps its old groups.
- **Pitfall: service accounts with login shells.** Don't give `www-data` or DB users `/bin/bash` "to debug" and forget — a service account that can log in is an attack surface. Use `sudo -u` to act as them instead.
- **Pitfall: `su alice` vs `su - alice`.** Without the `-`, you keep root's environment (PATH, HOME) — confusing. The dash gives a clean login environment.

## Checklist
- [ ] I can read `/etc/passwd` and `/etc/group` and explain UID 0, system vs human UID ranges.
- [ ] I can create/delete users and groups and inspect them with `id`/`groups`.
- [ ] I can add a user to a group with `usermod -aG` (and I know why the `a` matters).
- [ ] I can use `sudo` for admin and `sudo -u` to act as another user (e.g. `www-data`).
- [ ] I understand why services run as no-login accounts and why I shouldn't live as root.

## Resources
- `man useradd`, `man usermod`, `man sudo`, `man sudoers`
- Users & groups (Ubuntu): https://ubuntu.com/server/docs/user-management
- sudo / visudo guide: https://www.sudo.ws/docs/man/sudoers.man/
- `/etc/passwd` & `/etc/shadow` format: https://man7.org/linux/man-pages/man5/passwd.5.html
