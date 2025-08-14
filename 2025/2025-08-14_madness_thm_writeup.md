# MADNESS (TryHackMe) — Complete Walkthrough
**Date:** 2025-08-14  
**Category:** Web, Steganography, PrivEsc (Linux)  
**Perceived difficulty:** Medium

## Scenario / Context
This write-up documents my approach to solving the **MADNESS** room on TryHackMe. The objective is to obtain the user flag (`user.txt`) and then escalate privileges to read the root flag (`/root/root.txt`). Where third‑party tools are used, they are listed transparently—this document is focused on **method, reasoning, and lessons learned**.

## Environment
- **Attacker:** Linux workstation (Kali/Parrot‑like), terminal + browser
- **Target:** TryHackMe “MADNESS” instance (ephemeral IP)
- **Notes:** Replace any real IP/host with placeholders (e.g., `10.10.10.10` / `example.com`).

## Tools used (third‑party)
- `nmap` — network/port scanner
- `gobuster` — directory enumeration
- `exiftool` — image metadata viewer
- `hexedit` (or `xxd`) — inspect/modify file headers (magic numbers)
- `Burp Suite` (Intruder) — parameter brute forcing
- `steghide` — extract embedded data from images
- `CyberChef` — quick transforms (e.g., ROT13)
- `ssh` — remote shell access
- `find` — privilege escalation reconnaissance (SUID search)
- Exploit from `exploit‑db` related to `screen 4.5.0`
- Text editor (`nano`/`vim`) for quick edits

## Recon
Initial port scan against the target IP:
```bash
nmap -sS -A <TARGET_IP>
```
**Findings:**
- **22/tcp (ssh)** — open
- **80/tcp (http)** — open

Given SSH is open but credentials are unknown, pivot to the web service on port 80.

## Web analysis & content discovery
1. Browse `http://<TARGET_IP>/` and review the page. A top‑right image fails to load; viewing source reveals a suspicious HTML comment (e.g., “They will never find me”) suggesting hidden content.
2. Run directory brute forcing:
   ```bash
   gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt -t 50 -x php,txt
   ```
   (No valuable paths discovered; manual source review remains key.)

### Image anomaly & magic number
- Download the broken image (`thm.jpg`) and inspect:
  ```bash
  exiftool thm.jpg
  xxd -l 16 thm.jpg
  ```
- The file extension says **.jpg** but the **magic number** identifies it as a **PNG** (header mismatch). Fix the header to proper JPEG (example JPEG header bytes):
  ```
  FF D8 FF E0 00 10 4A 46 49 46 00 01
  ```
  After correcting the header with `hexedit`, the image renders and reveals a **hidden directory** path. Visiting it and re‑checking page source exposes another hint: a parameterized endpoint like:
  ```
  http://<TARGET_IP>/<HIDDEN_PATH>/?secret=<0-99>
  ```

### Brute forcing the `secret` parameter
Use **Burp Suite Intruder** to sweep the numeric range:
- Position a payload on the `secret` value.
- Payload type: **Numbers** `0..99`.
- Launch attack; identify the outlier by **response length**. Visiting the “heavy” response reveals an alphanumeric **code**.

### Steganography: extracting the hidden file
- Use that code as the **passphrase** with `steghide` against `thm.jpg` to extract an embedded text file:
  ```bash
  steghide extract -sf thm.jpg  # Enter the discovered passphrase
  ```
- The extracted `hidden.txt` contains a **username** that looks like `wbxre`. Using **CyberChef** with **ROT13** decodes it to the actual username.
- For the **password**, download the other available image (the one shown on the main page) and try steghide **with an empty passphrase**, which reveals the password.

### Gaining user shell
With username + password obtained from stego steps:
```bash
ssh <USER>@<TARGET_IP>
cat ~/user.txt
```

## Local privilege escalation (to root)
1. Enumerate SUID binaries:
   ```bash
   find / -perm -4000 -type f 2>/dev/null
   ```
2. Among standard SUIDs, an unusual entry appears: **screen 4.5.0**.
3. Locate the known exploit for `screen 4.5.0` (exploit‑db). Following the procedure (placing the PoC on the box via text editor, making it executable, and running it) yields a **root shell**:
   ```bash
   # (on target as user)
   nano exploit.c  # or paste PoC steps provided by the advisory
   gcc exploit.c -o screenroot   # if compilation is needed
   ./screenroot
   id
   cat /root/root.txt
   ```

## Post‑ex / Evidence
- `user.txt` and `root.txt` captured (contents omitted).
- Key artifacts: the mismatched image header, the hidden directory + parameter, stego‑extracted credentials, SUID `screen 4.5.0` exploitation.

## Mitigations / Lessons learned
- **Web hygiene:** remove debug comments; avoid “security by obscurity” paths and numeric secrets.
- **File integrity:** ensure media files use correct headers; mismatches can leak clues.
- **Stego exposure:** do not publish production images that embed secrets, even innocently; sanitize assets.
- **Auth hardening:** never reuse weak/guessable credentials; enforce MFA where possible.
- **System hardening:** remove/patch vulnerable SUID binaries (`screen`), keep packages updated, apply least‑privilege.
- **Blue‑team note:** response‑size and timing anomalies are strong signals; normalize or pad responses to reduce side‑channel hints.

## References
- TryHackMe — **MADNESS** room (platform resource)
- Nmap, Gobuster, Burp Suite (official docs)
- `steghide` documentation
- CyberChef (ROT13)
- `screen 4.5.0` privilege‑escalation advisory on exploit‑db
