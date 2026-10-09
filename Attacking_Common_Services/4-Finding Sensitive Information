Finding Sensitive Information: Study Notes
Core idea

When attacking a service, you play detective: collect as much information as possible and observe details carefully. Every single piece of information matters, even things that look insignificant at first.
Worked example: the johnsmith chain

Goal: get Remote Code Execution (RCE) on any of email, FTP, databases, or storage.
Step 	Action 	Result
1 	Enumerate all services, try anonymous access everywhere 	Only FTP allows anonymous access
2 	Look inside FTP 	Find an empty file named johnsmith
3 	Try johnsmith:johnsmith on FTP 	Fails
4 	Try the same creds on email 	Succeeds — logged in
5 	Search mailbox for the word "password" 	Multiple hits
6 	One email contains John's MSSQL database credentials 	Access database
7 	Use MSSQL's built-in command execution functionality 	RCE achieved

Lesson: a misconfigured service (anonymous FTP) exposed something that looked insignificant — just a filename, johnsmith — but that single detail was the thread that led to full RCE. Nothing should be dismissed as unimportant during enumeration.
Categories of sensitive information to look for

    Usernames
    Email addresses
    Passwords
    DNS records
    IP addresses
    Source code
    Configuration files
    PII (Personally Identifiable Information)

Services this module focuses on

    File shares
    Email
    Databases

Two key elements to finding sensitive information

    Understand the service — how it works, what it stores, how it's normally used.
    Know what you're looking for — have the categories above in mind so you recognize value when you see it.

Why "understanding the target" matters

Every target is unique. Before you can know what's sensitive, you need to understand:

    The target's processes and procedures
    Their business model
    Their purpose

Only once you understand the target can you judge what information matters to them — and therefore what's useful to you as an attacker.
Pentester's quick checklist

When enumerating a service, ask:

    Does this service allow anonymous access?
    What files, records, or messages can I read — even empty or unremarkable-looking ones?
    Could any single piece of data (a filename, a username, a stray string) be reused as credentials elsewhere?
    Am I noting every username, email, IP, and config detail, not just the "obvious" finds?
    Have I searched accessible content (e.g., email) for keywords like password, config, key?
    Does this finding chain into another service?

One-line summary

    Small, seemingly unimportant details — a filename, a stray credential — are often the key that unlocks the next service. Treat every piece of information as potentially significant.
