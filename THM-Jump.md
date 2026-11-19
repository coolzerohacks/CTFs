# Jump — TryHackMe

**Platform:** TryHackMe · **Difficulty:** easy · **Date:** 2026-11-19

## Summary
A five-user privilege-escalation chain from an anonymous FTP login to root. Each hop
abuses a different flavour of "something trusted runs code you can write": an upload
pipeline, a group-writable timed script, a PATH hijack, a relative-path call inside a
sudo'd script, and finally a GTFOBins sudo binary. Foothold and every lateral move is a
variation on planting your SSH key into the next user's `authorized_keys`.

## Enumeration
`nmap -sV TARGET` → two ports open: 21 (FTP, anonymous allowed) and 22 (SSH).
Logged into FTP anonymously; `incoming/` is world-writable and an automated pipeline
executes scripts dropped there as `recon_user`.

## Foothold
Generated an SSH keypair on Kali, wrote a small `.sh` that appends the public key to
`recon_user`'s `authorized_keys`, and `put` it into `incoming/`. The pipeline ran it as
`recon_user`; `ssh -i jump_key recon_user@TARGET` → **flag 1**.

## Privilege escalation
Four hops, each the same key-plant idea against a different weakness:

1. **recon_user → dev_user** — `/opt/dev/backup.sh` is group `dev_user` and
   group-writable, run on a short timer as `dev_user`. Overwrote it to plant the key. → flag 2
2. **dev_user → monitor_user** — a healthcheck service running as `monitor_user` calls
   `ps` with `/opt/dev/bin` first on its `PATH`, and that dir is writable. Dropped a
   malicious `ps` there; the service ran it. → flag 3
3. **monitor_user → ops_user** — `sudo -l` shows `(ops_user) NOPASSWD:
   /usr/local/bin/deploy.sh`, which does `cd /opt/app && ./deploy_helper.sh` by relative
   path. `/opt/app/deploy_helper.sh` is writable; overwrote it and ran the sudo'd script. → flag 4
4. **ops_user → root** — `sudo -l` shows `(root) NOPASSWD: /usr/bin/less`. GTFOBins:
   `sudo /usr/bin/less /etc/profile` then `!/bin/sh` spawns a root shell. → root

## Flags / Answers
- flag 1 (recon_user): `REDACTED`
- flag 2 (dev_user): `REDACTED`
- flag 3 (monitor_user): `REDACTED`
- flag 4 (ops_user): `REDACTED`
- root flag (`/root/flag.txt`, not root.txt): `REDACTED`

## Lessons
- Every hop in a chain can be the *same* primitive (key-plant) against a different
  trust boundary — writable timer script, PATH hijack, relative-path sudo call, GTFOBins.
- `sudo -l` is the first thing to run on every new user. It answered two of the four hops.
- GTFOBins `less` → `!/bin/sh` is a must-know; `vim`, `find`, `awk` do the same.
