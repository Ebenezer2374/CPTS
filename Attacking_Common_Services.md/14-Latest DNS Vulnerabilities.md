Latest DNS Vulnerabilities: Study Notes (Subdomain Takeover)
Core idea

Companies cancel third-party services but forget to remove the DNS CNAME records pointing to them — because DNS entries cost nothing to keep. The result: a subdomain still points to a non-existent resource at a provider like AWS, GitHub, or Akamai. Anyone can register that orphaned resource and take control of the subdomain.

Scale of the problem (RedHuntLabs, 2020):

    220 million domains studied
    424,120 subdomains found vulnerable to takeover
    62% belonged to the e-commerce sector
    139 vulnerable subdomains in the Alexa Top 1000
    33 third-party services identified as prone to this vulnerability

Bug bounties: HackerOne explicitly lists subdomain takeover as its own bounty category, with significant payouts. Many automated discovery tools exist on GitHub for finding and PoC-ing these.
Why it's dangerous

A taken-over subdomain sits under the official company domain — e.g., customer-drive.inlanefreight.com. Visitors see the legitimate parent domain (inlanefreight.com) in the URL and trust it. The attacker controls what that page shows — a perfect phishing setup. Customers don't know the page was created or mirrored by an attacker.

Beyond phishing, subdomain takeover can also enable:

    Cookie stealing
    Cross-Site Request Forgery (CSRF)
    Abusing CORS (Cross-Origin Resource Sharing)
    Defeating Content Security Policy (CSP)

How it works (the mechanics)

    Company has a CNAME record: sub.target.com → anotherdomain.com
    The company stops using the third-party service at anotherdomain.com
    anotherdomain.com (or the resource behind it, like an S3 bucket) expires or is deleted
    Visiting sub.target.com returns a 404 / NoSuchBucket / NoSuchDistribution error
    Attacker registers the now-available anotherdomain.com (or claims the S3 bucket, GitHub Pages site, etc.)
    The CNAME in the company's DNS still points there — so sub.target.com now resolves to the attacker's content
    A single DNS change (the original CNAME, untouched by anyone) made the attacker the effective owner of that subdomain

The third-party provider is usually not responsible for checking whether their registrant should own the subdomain — that responsibility stays with the primary domain owner.
Mapped to the Concept of Attacks (two chained loops)
Loop 1: Initiation of Subdomain Takeover
# 	What happens 	Category
1 	The abandoned subdomain name discovered via enumeration is the Source 	Source
2 	Attacker registers the orphaned domain/resource on the third-party provider and links it to their own server 	Process
3 	Privileges here lie with the primary domain owner — who hasn't removed the CNAME; the third-party provider doesn't check who "should" own it 	Privileges
4 	The attacker's own server, now linked via the registration, is the Destination 	Destination
Loop 2: Trigger the Forwarding
# 	What happens 	Category
5 	A visitor types the subdomain URL — the outdated CNAME record (still in the company's DNS) acts as the Source 	Source
6 	The DNS server looks up the subdomain, finds it in its records, and redirects the user to the corresponding destination — which is now attacker-controlled 	Process
7 	The DNS server trusts the subdomain because it's in the zone's records, managed by the legitimate domain's admins who never removed it 	Privileges
8 	The visitor (requesting the IP of the subdomain) is the final Destination — landing on the attacker's page 	Destination

Pattern recap: same two-loop structure as the other CVE examples. Loop 1's Destination (attacker's registered resource) becomes loop 2's Process outcome (where DNS forwards real visitors).
How to find vulnerable subdomains

Enumerate subdomains (Subfinder, Sublist3r, Subbrute — see Attacking DNS note)

Check CNAME records:

host support.inlanefreight.com

Example output: support.inlanefreight.com is an alias for inlanefreight.s3.amazonaws.com

Confirm the resource is unclaimed: browse the subdomain — look for provider-specific error pages:

    AWS S3: NoSuchBucket
    GitHub Pages: There isn't a GitHub Pages site here
    Fastly: Fastly error: unknown domain
    Heroku: No such app

Reference: can-i-take-over-xyz (GitHub) — lists vulnerable services and how to verify/exploit each one.
Pentester's quick checklist

    Enumerate all subdomains of the target domain
    For each, check for CNAME records pointing externally (host <sub> or dig CNAME <sub>)
    Resolve the CNAME destination — is it active or returning an error?
    Match the error type to a known provider's "unclaimed resource" error
    Verify via can-i-take-over-xyz whether that provider is exploitable
    Register/claim the resource at the provider and confirm the subdomain resolves to your content
    Document fully for the report or bug bounty submission (CNAME record, evidence of unclaimed resource, PoC page)

One-line summary

    Subdomain takeover exploits forgotten CNAME records — once a company stops using a third-party resource but leaves the DNS entry, anyone who claims that resource becomes the silent owner of the subdomain, with full control over what visitors see.
