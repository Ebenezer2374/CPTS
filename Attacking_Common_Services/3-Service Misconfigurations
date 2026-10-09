Service Misconfigurations: Study Notes
Core idea

A misconfiguration happens when an admin, support person, or developer doesn't correctly configure the security of an app, site, desktop, or server, leaving open pathways for unauthorized users. It's a category in the OWASP Top 10 (Security Misconfiguration).
The four main types
1. Authentication

    Default credentials: common on older apps and some vendors. Admins often never change them.
    Weak or blank passwords: admins set these "temporarily" during setup and forget to change them.
    Pentest workflow: grab the service banner → look up default credentials for that product → if none, try weak combos.

Weak combos to try:

admin:admin
admin:password
admin:<blank>
root:12345678
administrator:Password

Defender side: enforce a password policy with minimum complexity.
2. Anonymous Authentication

The service allows anyone with network access to connect without being prompted for credentials. Always test for it (e.g., anonymous FTP login).
3. Misconfigured Access Rights

Accounts have more permissions than their role needs.

    Example: a user whose job is only to upload files to an FTP server is also allowed to read everything.
    What you might find as a result: config files for other services, plaintext credentials, usernames, proprietary data, PII.
    Bigger picture: lower-level staff can reach data meant only for managers or admins.
    Defender side: plan an access strategy, using RBAC (role-based access control) or ACLs (access control lists).

4. Unnecessary Defaults

Out-of-the-box settings, features, files, and credentials are designed for usability, not security. Leaving them in production expands the attack surface.

Attackers can often get default creds or exploit weak settings with a quick internet search.

OWASP Top 10 items related to defaults:

    Unnecessary features enabled or installed (ports, services, pages, accounts, privileges)
    Default accounts and passwords still enabled and unchanged
    Error handling that reveals stack traces or overly informative errors
    Upgraded systems with the latest security features disabled or insecurely configured

Preventing misconfiguration

Principle: lock down critical infrastructure and only allow desired behavior. Disable any communication the program doesn't need.

Checklist:

    Disable admin interfaces
    Turn off debugging
    Disable default usernames and passwords
    Prevent unauthorized access and directory listing
    Run regular scans and audits to catch future misconfigurations or missing fixes

OWASP guidance for secure installation:

    Repeatable, automated hardening process: dev, QA, and production configured identically, but with different credentials in each
    Minimal platform: remove or don't install unused features, components, docs, and samples
    Review configs against security notes, updates, and patches as part of patch management; review cloud storage permissions (e.g., S3 buckets)
    Segmented architecture: segmentation, containerization, or cloud security groups (ACLs)
    Send security directives to clients (e.g., security headers)
    Automated verification that configurations work in all environments

Pentester's quick checklist

When you meet a service, ask:

    Can I connect anonymously?
    What's the banner/version, and are there default creds for it?
    Do weak combos work?
    Once in, what can this account read/write beyond its role?
    Are there leftover defaults (debug pages, sample files, admin interfaces, directory listing, verbose errors)?
    What sensitive data is exposed (configs, plaintext creds, PII)?

One-line summary

    Most misconfigurations come from defaults left unchanged, weak or missing authentication, and permissions broader than needed. Attackers check these first because they're easy wins.
