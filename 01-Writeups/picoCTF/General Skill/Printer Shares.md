---
platform: picoCTF
category: General Skill
difficulty: Easy
points: 
date: 
tags:
  - ctf
  - general_skill
solved: true
---
# Challenge Name: Printer Shares

## Challenge Description
![](../../../05-Assets/Pasted%20image%2020260523152101.png)
## Initial Analysis
- As described in the challenge, a file has been sent to the printer, and it asks if we can access the file from that printer.
- The hint here tells us that the smb (server message block) and smbclient (Linux) protocols will help us connect to the printer.
## Solution
### Step 1: Check connect
- First, we check if the server port was open using netcat:

```bash
$ nc -vz mysterious-sea.picoctf.net <port>
```
- Output:

![](../../../05-Assets/Pasted%20image%2020260523154243.png)
- Port open => The printer service was online
### Step 2: List SMB shares
- In this context, "shares" can also be understood as a folder that the server publishes to the network, allowing other computers to access it freely, just like a local file on their own machine.
- We using smbclient to enumerate shares:
```bash
smbclient -L //mysterious-sea.picoctf.net -p <port> -N
```
- Output:

![](../../../05-Assets/Pasted%20image%2020260523160040.png)
=>This showed the correct share name: shares.
### Step 3: Connect to the public shares
- We connect to the share using:
```bash
smbclient //mysterious-sea.picoctf.net/shares -p <port> -N
```
- We reached the SMB prompt:
```bash
smb: \>
```
### Step 4: List Files on the Share
- Inside the share, we listed the files:
```bash
ls
```
- Ouput

![](../../../05-Assets/Pasted%20image%2020260523160333.png)
### Step 5: Download and Read the Flag
- We retrieved the flag file:
```bash
get flag.txt
exit
```
**We are forced to retrieve the file to our local machine to read it because we cannot read the file within the smbclient.**
- Read file
```bash
cat flag.txt
```
- Output:

![](../../../05-Assets/Pasted%20image%2020260523160822.png)
## Flag
`FLAG{REDACTED}`

## Tools Used

| Tool       | Purpose                                                            | Cheet sheet |
| ---------- | ------------------------------------------------------------------ | ----------- |
| nc(netcat) | Used to check if the service is running and if the port is open. | [nc(netcat)-command-cheetsheet](../../../03-Tools/nc(netcat)-command-cheetsheet.md)            |
| smbclient           | Used to access the SMB protocol.                                                                 |[smbclient-command-cheetsheet](../../../03-Tools/smbclient-command-cheetsheet.md)             |

## Takeaway
- This challenge simulates how a misconfigured printer network can be exploited via smb.
- Using smbclient allows us to access the public share of the smb protocol anonymously.
- nc allows us to verify if ports are still active but cannot connect to the smb protocol.
- Always try to list potential shares using -L before attempting an exploit.
