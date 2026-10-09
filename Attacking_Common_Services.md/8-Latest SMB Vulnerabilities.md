Latest SMB Vulnerabilities: Study Notes (SMBGhost — CVE-2020-0796)
Core idea

SMBGhost (CVE-2020-0796) is a major vulnerability in SMB's compression mechanism (SMB v3.1.1), affecting Windows 10 versions 1903 and 1909. It let an unauthenticated attacker achieve remote code execution (RCE) and full access to the target.

This note covers the concept, not deep exploit-dev detail — understanding the underlying bug (integer overflow) requires reverse engineering, CPU, kernel, and exploit-dev knowledge, which is out of scope here. The point: even a complex vuln like this still fits the same Source → Process → Privileges → Destination pattern.
What kind of bug is this?

An integer overflow in an SMB driver function, which allows system commands to be overwritten while accessing memory.

What's an integer overflow? The CPU tries to produce a number larger than the allocated memory space can hold. Example: if a variable is never supposed to go negative, but an arithmetic operation produces a negative result, and that negative value gets wrongly stored/returned as a positive integer — unexpected behavior follows.

Root cause here: the function handling SMB session negotiation data size lacked proper bounds checks.

(For the underlying mechanics of buffer manipulation, see the dedicated Stack-Based Buffer Overflow modules — Linux x86 and Windows x86.)
Where it happens

The bug triggers while processing a malformed compressed message, sent right after the Negotiate Protocol Response.

    SMB server listens on TCP/445
    If compression is supported, client and server agree on communication terms before more data is sent
    If the data sent exceeds the integer variable's limits, the excess gets written into the buffer
    This overwrites subsequent CPU instructions, interrupting normal program execution
    An attacker can craft the data so the overwritten instructions become their own instructions — hijacking what the CPU executes next

Mapped to the Concept of Attacks (two chained loops)
Loop 1: Initiation of the Attack
# 	What happens 	Category
1 	Client sends an attacker-manipulated request to the SMB server 	Source
2 	Compressed packets are processed per the negotiated protocol responses 	Process
3 	Processing runs with system/administrator privileges 	Privileges
4 	The local process meant to handle these compressed packets is the Destination 	Destination
Loop 2: Trigger Remote Code Execution
# 	What happens 	Category
5 	The Source for this second loop is the output of the previous process 	Source
6 	Integer overflow occurs — the overwritten buffer now contains the attacker's own instructions, which the CPU executes 	Process
7 	Same SMB server privileges as before apply 	Privileges
8 	The remote attacker's system is the Destination — they gain access to the local target 	Destination

Pattern recap: same structure as Log4j and CoreFTP — the output of loop 1 (a corrupted buffer ready for hijacking) becomes the input of loop 2 (actual code execution), and loop 2's Destination is control handed back to the attacker.
Why this example matters

Even though the real exploit requires manipulating a buffer at a low level (seen in the PoC), the high-level attack concept is unchanged from simpler examples like Log4j or CoreFTP. The four-category pattern scales from "misinterpreted HTTP header" all the way up to "kernel-level integer overflow."
Pentester's quick checklist

    Identify SMB version — check if SMB v3.1.1 compression is in use
    Check Windows build — 1903 / 1909 are the vulnerable versions
    Confirm SMB is reachable on TCP/445 with no auth required (unauthenticated vuln)
    Note: this is a kernel-level bug — exploitation risk includes system crashes (BSOD) as well as RCE; treat PoCs with caution on production systems
    Patch status is the real defensive question — check if MS has issued the fix for the target's build

One-line summary

    SMBGhost shows that even a deep, kernel-level integer-overflow RCE still follows the same Source → Process → Privileges → Destination pattern — just with the "Process" step replaced by low-level memory corruption instead of a misread string.
