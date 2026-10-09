Latest SQL Vulnerabilities: Study Notes (xp_dirtree Hash Theft)
Core idea

This isn't a CVE and needs no direct exploit code — it's an abuse of legitimate functionality. It's the same xp_dirtree NTLMv2-hash-stealing technique from the "Attacking SQL Databases" note, explained here as a worked Concept-of-Attacks example.

Two variants exist:

    Direct connection to the MSSQL server (the focus here — simpler)
    Through a vulnerable web application (not covered in detail here)

What xp_dirtree actually is

An undocumented MSSQL function that lists the contents of a folder — local or remote. Accepts parameters including:

    Depth — how far into subfolders to recurse
    Target folder — local or remote (UNC) path

Why it leaks a hash

xp_dirtree isn't itself "vulnerable" — it abuses how SMB authentication works. When a Windows host tries to access a shared folder over the network, it automatically sends an NTLMv2 hash to authenticate. Point xp_dirtree at a folder you control, and the MSSQL service account's hash comes straight to you.
What you can do with the captured hash

    Crack it locally → recover the cleartext password
    SMB Relay attack → "replay" the hash to log into other systems where the account has local admin

Important limitation: Microsoft patched the old flaw that let you relay a hash back to the same host it came from. So a successful relay typically gets you admin on a different host — but from there you could steal further credentials, which might eventually let you circle back to the original system.
Mapped to the Concept of Attacks (two chained loops)
Loop 1: Initiation of the Attack
# 	What happens 	Category
1 	User input specifies the function (xp_dirtree) and the network share to target 	Source
2 	Process ensures the contents of the specified folder get listed/displayed 	Process
3 	Executing this on the MSSQL server requires the elevated privileges the SQL service runs with 	Privileges
4 	The SMB service is the Destination the request gets forwarded to 	Destination
Loop 2: Steal the Hash
# 	What happens 	Category
5 	The SMB service receives the forwarded request as its new Source (from the MSSQL process in loop 1) 	Source
6 	SMB processes the request, querying the specified folder's contents 	Process
7 	The MSSQL service account's authentication hash is used, since that's the account making the SMB query 	Privileges
8 	Destination: the host you control, where the shared folder (and the authentication attempt) lands 	Destination

Pattern recap: same structure as Log4j, CoreFTP, and SMBGhost — the Destination of loop 1 (the SMB service being contacted) becomes the Source of loop 2 (SMB now processing an authentication request that leaks the hash to you).
Capturing the hash

Once the MSSQL service account tries to authenticate to your controlled share, tools like Responder, Wireshark, or tcpdump intercept and display the NTLMv2 hash for you to use.

(See the "Attacking SMB" and "Attacking SQL Databases" notes for the exact xp_dirtree/xp_subdirs commands and Responder/impacket-smbserver setup.)
Beyond this technique

MSSQL offers many other command-execution routes — e.g., running Python code inside a SQL query (per Microsoft's documentation). These and other MSSQL attack possibilities are covered in a separate module.
Pentester's quick checklist

    Confirm you have some level of access to run queries on the MSSQL server (direct connection or via a vulnerable web app)
    Stand up a listener first — Responder or impacket-smbserver
    Run xp_dirtree (or xp_subdirs) pointed at your listener's share
    Capture the NTLMv2 hash as it comes in
    Decide: crack it locally, or relay it — remembering relay-back-to-origin is patched, so target a different host
    If relay succeeds, look for credential reuse opportunities to eventually circle back to the original system

One-line summary

    xp_dirtree isn't a bug in MSSQL — it's a free hash-leak because any Windows host, including the SQL service account, auto-authenticates to SMB shares you point it at; capture that hash, then crack or relay it.
