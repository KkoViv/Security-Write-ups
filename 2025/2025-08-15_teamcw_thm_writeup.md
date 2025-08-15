# TryHackMe - Team CW
**Date:** 2025-08-15  
**Category:** Web / PrivEsc  
**Perceived difficulty:** Medium

## Scenario / Context
This TryHackMe CTF-style room demonstrates how poor confidentiality of critical data, combined with a Local File Inclusion (LFI) vulnerability, can be chained to gain initial access to a Linux machine and escalate privileges to root.

## Environment
- **OS / VM:** TryHackMe-hosted Linux target
- **Browser / Terminal:** Firefox, Burp Suite integrated browser
- **Network notes / service version:** Apache HTTP server (default page on port 80), FTP service (port 21), SSH service (port 22)

## Tools used (third-party)
- `nmap 7.94`
- `gobuster 3.5`
- FTP client
- Burp Suite Community Edition
- SecLists wordlists

> **Note:** These are not my tools; this write-up documents method and reasoning.

## Recon
Initial port scan:
```bash
nmap -sS -p- <TARGET_IP>
```
Findings:
- **21/tcp** – FTP (file sharing; no anonymous login allowed)
- **22/tcp** – SSH (requires credentials)
- **80/tcp** – HTTP (Apache default page)

FTP anonymous login attempt failed.

Inspecting the Apache default page source code reveals a developer comment pointing to another virtual host: `team.thm`. Added this to `/etc/hosts`.

First `gobuster` scan on the IP site:
```bash
gobuster dir -u http://<TARGET_IP> -w <wordlist>
```
→ No useful results.

Second `gobuster` scan on `team.thm`:
```bash
gobuster dir -u http://team.thm -w <wordlist>
```
- `/robots.txt` → contains the string `dale` (possible username)
- `/assets`, `/scripts` (not directly viewable)
- `/images` (site images)

Enumerating `/scripts` with extensions:
```bash
gobuster dir -u http://team.thm/scripts -w <wordlist> -x txt,old,bak,new
```
- `script.txt` → note about older version containing FTP credentials
- `script.old` → contains valid FTP credentials

## Exploitation / Analysis
1. **FTP Access**  
   Logged in with recovered credentials.  
   Downloaded `New_site.txt` from `/workshare`, revealing:
   - Another vhost: `dev.team.thm`
   - SSH `id_rsa` keys might be stored in system config files.

2. **dev.team.thm Enumeration**  
   Added to `/etc/hosts`.  
   Site contains link to:
   ```
   http://dev.team.thm/script.php?page=teamshare.php
   ```
   Testing with `/etc/passwd` confirms **LFI vulnerability**.

3. **LFI Exploitation**  
   Using Burp Suite Sniper with SecLists wordlist to replace the `page` parameter with common Linux file paths.  
   Discovered `/etc/ssh/sshd_config` containing `id_rsa` for user `dale`.

4. **SSH Access as dale**  
   Saved key to `dale.key` and set proper permissions:
   ```bash
   chmod 600 dale.key
   ssh -i dale.key dale@<TARGET_IP>
   ```
   Retrieved `user.txt`.

5. **Privilege Escalation to gyles**  
   Checked sudo privileges:
   ```bash
   sudo -l
   ```
   → dale can run `/home/gyles/admin_checks` as user gyles without password.  
   Script reads input into a variable and executes it unsafely:
   ```bash
   $error 2>/dev/null
   ```
   Exploit:
   ```bash
   sudo -u gyles /home/gyles/admin_checks
   # When prompted, enter: /bin/bash
   ```
   Now shell as gyles. Upgraded TTY:
   ```bash
   python3 -c 'import pty; pty.spawn("/bin/bash")'
   ```

6. **Privilege Escalation to root**  
   Exploring `/opt/admin_stuff` revealed `script.sh` referencing `main_backup.sh` (owned by root but writable by gyles).  
   Modified it to:
   ```bash
   chmod +s /bin/bash
   ```
   After one minute (cron job run), gained root shell:
   ```bash
   /bin/bash -p
   ```
   Retrieved `/root/root.txt`.

## Post-ex / Evidence
- **Username:** dale (SSH)
- **User flag:** `user.txt`
- **Root flag:** `root.txt`

## Mitigations / Lessons learned
- Remove or restrict access to unused virtual hosts.
- Avoid storing credentials in scripts or backups (`.old`, `.bak`).
- Validate and sanitize user input to prevent LFI.
- Restrict file permissions; do not make root-owned scripts writable by other users.
- Limit sudo permissions to necessary commands only.

## References
- [TryHackMe - Team CW](https://tryhackme.com/room/teamcw)
- [OWASP - Local File Inclusion](https://owasp.org/www-community/attacks/Path_Traversal)
