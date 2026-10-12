Latest Email Service Vulnerabilities: Study Notes (OpenSMTPD — CVE-2020-7247)
Core idea

CVE-2020-7247 — a critical RCE vulnerability in OpenSMTPD up to version 6.6.2, publicly disclosed in 2020 but exploitable since 2018. Affects many Linux distributions: Debian, Fedora, FreeBSD, and others.

Two key dangers:

    Allows remote code execution on the server
    Requires no authentication to trigger

Scale: as of April 2022, 5,000+ publicly accessible OpenSMTPD servers on Shodan, with numbers growing. At the time the data was recorded, the US (1,305), Germany (1,187), and France (553) had the most exposed instances.
What kind of bug is this?

The vulnerability lives in the function that records the sender's email address. By inserting a semicolon (;) into the sender field, an attacker can escape the function and inject arbitrary shell commands.

Constraint: there is a 64-character limit on what can be inserted as a command — so payloads need to be short (e.g., a short reverse shell or a wget/curl stager).

(Full technical details are on Exploit-DB; an exploit is also published there for analysis.)
Mapped to the Concept of Attacks (two chained loops)
Loop 1: Initiation of the Attack
# 	What happens 	Category
1 	Attacker connects to SMTP and composes an email — the crafted input (sender field with injected command) is the Source 	Source
2 	The service takes the email and its fields (sender, recipient, body) for processing 	Process
3 	Listening on standard port 25 requires root; the service runs with elevated privileges accordingly 	Privileges
4 	The entered data is forwarded to another local process for further handling 	Destination
Loop 2: Trigger Remote Code Execution
# 	What happens 	Category
5 	The entire input — particularly the sender field containing the injected command after ; — becomes the new Source 	Source
6 	The process reads the input; the semicolon triggers a special code path that breaks out of normal parsing and executes the shell command 	Process
7 	Since OpenSMTPD already runs with elevated privileges, the injected command inherits those same privileges 	Privileges
8 	Destination: the shell command calls back over the network to the attacker's host, establishing remote access 	Destination

Pattern recap: same two-loop structure as Log4j, CoreFTP, SMBGhost, BlueKeep, and xp_dirtree — a crafted input (loop 1) seeds a second cycle where the payload executes (loop 2) and the result reaches the attacker.
Pentester's quick checklist

    Identify OpenSMTPD on port 25 via Nmap (-sV will reveal the banner/version)
    Confirm the version is ≤ 6.6.2
    No credentials needed — connect directly and compose the malicious email
    Craft the sender field: attacker@domain.com; <command> — keep the command under 64 characters
    Use a short stager if needed (e.g., curl or wget to pull a script, then execute it)
    Check Exploit-DB for the published PoC to understand exact formatting before running it
    Catch the callback with nc -lvnp <port>

HTB boxes to practice email attacks on
Box 	What it covers
Rabbit 	Brute-forcing Outlook Web Access (OWA), then phishing via malicious macro document
SneakyMailer 	Phishing + enumerating a user's inbox with Netcat and an IMAP client
Reel 	SMTP user brute-force + phishing with a malicious RTF file

Tip: use ippsec.rocks to search for any attack technique by keyword — it maps terms to the HTB boxes where that technique appears, giving you a list of targets to practice on.
One-line summary

    OpenSMTPD's sender-field parser can be escaped with a semicolon, letting an unauthenticated attacker inject shell commands that run with root-level privileges — 64-character limit and all, it's enough for a reverse shell stager.
