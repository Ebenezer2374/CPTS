Attacking DNS: Study Notes
Core idea

DNS translates domain names (e.g., hackthebox.com) to IP addresses (e.g., 104.17.42.72). Runs mainly on UDP/53, falling back to TCP/53 when a response is too large for a single UDP packet (and zone transfers always use TCP). Because nearly every network app relies on DNS, attacks against DNS servers are among the most prevalent and significant threats today.
Enumeration

DNS data reveals how an organization operates — services, third parties, email providers, etc. (see the Footprinting module's Domain Information section).

nmap -p53 -Pn -sV -sC 10.10.110.213

Reveals the server software/version (e.g., ISC BIND 9.11.3-1ubuntu1.2).
DNS Zone Transfer

A DNS zone is the portion of the namespace a specific org/admin manages. Servers replicate zone data via zone transfers. No authentication is required by default — only correctly restricting which IPs can request a transfer prevents abuse. Transfers use TCP even though normal DNS queries use UDP.

Impact: an attacker can dump an org's entire internal DNS namespace, expanding the attack surface (internal hostnames, subdomains, etc.).
dig — AXFR zone transfer

dig AXFR @ns1.inlanefreight.htb inlanefreight.htb

Dumps all records: SOA, NS, A records for the domain and its hosts (e.g., admin., hr., support. subdomains with internal IPs).
Fierce — automated zone transfer check

fierce --domain zonetransfer.me

Enumerates the domain's nameservers and attempts a zone transfer against each, dumping found records (A, TXT, MX, NS, SRV, PTR, AFSDB, LOC, NAPTR, AAAA, etc.) if successful.
Domain Takeovers & Subdomain Enumeration

Domain takeover: registering an expired domain to gain control of it — useful for hosting malicious content or phishing under a trusted-looking name.

Subdomain takeover: same idea, applied via CNAME records. A subdomain often points to a third-party service (AWS, GitHub, Akamai, Fastly, etc.):

sub.target.com.   60   IN   CNAME   anotherdomain.com

If anotherdomain.com expires, anyone can register it and gain full control of sub.target.com until the DNS record is updated.
Subdomain enumeration tools

Subfinder — scrapes subdomains from open sources (e.g., DNSdumpster):

./subfinder -d inlanefreight.com -v

Sublist3r — can brute-force using a wordlist (mentioned as an alternative).

Subbrute — pure DNS brute-forcing with self-defined resolvers; useful on internal pentests where hosts lack internet access:

git clone https://github.com/TheRook/subbrute.git
cd subbrute
echo "ns1.inlanefreight.com" > ./resolvers.txt
./subbrute.py inlanefreight.com -s ./names.txt -r ./resolvers.txt

(Tip: use plenty of resolvers in resolvers.txt — fewer than 16 per process triggers a warning.)

Getting tools onto an internal/air-gapped host: physical access via USB, or working from a host already reached through pivoting.
Checking subdomains for takeover

host support.inlanefreight.com

Example: support.inlanefreight.com is an alias for inlanefreight.s3.amazonaws.com — if browsing it returns a NoSuchBucket error, the S3 bucket doesn't exist and you can claim it yourself, taking over the subdomain.

Reference: the can-i-take-over-xyz GitHub repo lists which services/providers are vulnerable to this pattern and how to assess them.
DNS Spoofing (DNS Cache Poisoning)

Altering legitimate DNS records (or responses) with false info to redirect traffic to a fraudulent destination. Two broad paths:

    MITM — intercept client-to-DNS-server traffic and respond with a fake answer
    Compromise the DNS server itself and modify its records directly

Local DNS Cache Poisoning (Ettercap)

1. Define the spoof mapping in /etc/ettercap/etter.dns:

inlanefreight.com      A   192.168.225.110
*.inlanefreight.com    A   192.168.225.110

2. Scan for live hosts: Ettercap → Hosts → Scan for Hosts.

3. Set MITM targets:

    Target1 = victim IP (e.g., 192.168.152.129)
    Target2 = default gateway IP (e.g., 192.168.152.2)

4. Enable the plugin: Plugins → Manage Plugins → dns_spoof — sends the victim spoofed DNS responses resolving inlanefreight.com to the attacker's IP.

Result: the victim browsing inlanefreight.com lands on the attacker's fake page; pings resolve to the spoofed IP too:

ping inlanefreight.com
Pinging inlanefreight.com [192.168.225.110] ...

Pentester's quick checklist

    nmap -p53 -sV -sC <target> — fingerprint the DNS server
    Attempt a zone transfer: dig AXFR @<ns> <domain> or run fierce
    Enumerate subdomains (subfinder, sublist3r, subbrute)
    For each subdomain with a CNAME to a third-party service, check if that external resource is unclaimed (e.g., NoSuchBucket, NoSuchDistribution) → potential subdomain takeover
    On an internal engagement with network position (MITM-capable), consider DNS spoofing with Ettercap/Bettercap to redirect specific domains
    Always treat found internal hostnames/IPs from zone transfers as high-value recon data (feed into further enumeration)

One-line summary

    DNS attacks split into two families: passive/structural (zone transfers, subdomain enumeration → takeover via dangling CNAMEs) and active/network-position (DNS spoofing via MITM) — both expand your view of, or control over, the target's infrastructure.

Question 1
Find all available DNS records for the "inlanefreight.htb" domain on the target name server and submit the flag found as a DNS record as the answer.
> dig ns inlanefreight.htb  @10.129.203.6
ns.inlanefreight.htb
> echo "ns.inlanefreight.htb" > ./resolvers.txt
> python3 subbrute.py inlanefreight.htb -s names.txt -r resolvers.txt
hr.inlanefreight.htb
> dig AXFR @inlanefreight.htb hr.inlanefreight.htb
HTB{LUIHNFAS2871SJK1259991}