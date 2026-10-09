Latest RDP Vulnerabilities: Study Notes (BlueKeep — CVE-2019-0708)
Core idea

BlueKeep (CVE-2019-0708), published 2019, is a critical RDP (TCP/3389) vulnerability leading to RCE. Fueled many malware/ransomware campaigns, especially against organizations (e.g., hospitals) running legacy, hard-to-maintain software/library versions.

Key danger: no prior access or authentication needed — it triggers during the initial connection setup, before login.

Again: this note focuses on the concept, not deep exploit-dev detail.
What kind of bug is this?

A Use-After-Free (UAF) vulnerability: the exploit frees a piece of memory, then the attacker writes data into that freed memory and tricks the CPU into executing it as code.

Where it triggers: right after connection initialization, during the basic settings exchange between client and server — before any authentication happens.

The deep kernel-level steps (how the memory gets freed, how the overwrite is staged) aren't needed to grasp the concept — see the linked technical analysis for that depth.
Mapped to the Concept of Attacks (two chained loops)
Loop 1: Initiation of the Attack
# 	What happens 	Category
1 	Attacker manipulates the initialization request during the client-server settings exchange 	Source
2 	The request reaches a function that creates a virtual channel — this function contains the vulnerability 	Process
3 	Because this service administers the system, it runs with LocalSystem privileges automatically 	Privileges
4 	The manipulated function redirects execution into a kernel process 	Destination
Loop 2: Trigger Remote Code Execution
# 	What happens 	Category
5 	Source: the attacker's payload, designed to free kernel memory and plant instructions there 	Source
6 	The kernel process frees the memory and the CPU ends up pointing at the attacker's code 	Process
7 	Since the kernel runs with the highest possible privileges, the planted instructions execute as LocalSystem 	Privileges
8 	Execution of the attacker's instructions sends a reverse shell back over the network 	Destination

Pattern recap: same structure as Log4j, CoreFTP, and SMBGhost — loop 1's Destination (a hijacked kernel process) becomes loop 2's Source (a payload freeing memory for code execution), ending in a reverse shell to the attacker.
Real-world impact

    Not all newer Windows versions are vulnerable (per Microsoft)
    Patches exist for supported versions, and Microsoft even patched several unsupported/legacy versions given the severity
    May 2019 scan: ~950,000 Windows systems found vulnerable
    Still today: roughly a quarter of those originally-identified hosts remain unpatched

Operational caution

BlueKeep is a flaw you'll likely encounter on engagements, but the exploit can cause system instability, including a full Blue Screen of Death (BSoD).

Best practice: if in doubt, talk to the client first — make sure they understand the crash risk before deciding whether to run the exploit.
Pentester's quick checklist

    Identify RDP on TCP/3389 and the Windows version/build
    Check if the build falls in BlueKeep's vulnerable range (pre-patch, older Windows)
    Remember: no credentials needed to trigger — but also no safe "test" mode — running the exploit risks crashing the host
    Get client sign-off before attempting exploitation given the BSoD risk
    If cleared to proceed, note the expected outcome: a reverse shell with LocalSystem privileges

One-line summary

    BlueKeep lets an unauthenticated attacker trigger a kernel-level Use-After-Free during RDP's initial handshake, escalating straight to a LocalSystem reverse shell — powerful, but risky enough to crash the target, so get client approval first.
