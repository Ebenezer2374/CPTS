HTB Academy Notes: HTTP & cURL
Study notes from the Web Requests module (HTTP basics and cURL intro).
1. What HTTP Is
• HTTP (HyperText Transfer Protocol) is an application-level protocol used to access World Wide Web resources.
• Most internet communication, for both web and mobile apps, happens through HTTP web requests.
• Hypertext means text containing links to other resources.
• Model: a client requests a resource, the server processes the request and returns the resource.
• Default port is 80 (HTTPS is 443). The port can be changed depending on web server configuration.
• We reach sites by entering an FQDN (Fully Qualified Domain Name) as part of a URL, e.g. www.hackthebox.com.
2. Anatomy of a URL
Example: http://admin:password@inlanefreight.com:80/dashboard.php?login=true#status
Component
Example
What it does
Scheme
http:// https://
Identifies the protocol the client uses. Ends with ://
User Info
admin:password@
Optional credentials (user:pass), separated from host by @
Host
inlanefreight.com
Resource location: a hostname or an IP address
Port
:80
Separated from host by :. Defaults: http = 80, https = 443
Path
/dashboard.php
The file or folder being accessed. If none given, server returns the default index (e.g. index.html)
Query String
?login=true
Starts with ?. Parameter plus value. Multiple parameters are separated by &
Fragment
#status
Handled client-side by the browser to jump to a section of the page
Key point: only the scheme and host are mandatory. Everything else is optional.
3. HTTP Flow (What Happens When You Visit a Site)
1. User enters a URL (e.g. inlanefreight.com) in the browser.
2. Browser checks the local /etc/hosts file first for a matching record.
3. If not found, the browser sends a DNS query to resolve the domain to an IP address (e.g. 152.153.81.14). Every domain must be resolved, since a server can't be reached without an IP.
4. Browser sends a GET request to the default HTTP port (80) asking for the root path /.
5. Web server processes the request. By default it returns an index file for /.
6. Server sends back the HTTP response (e.g. contents of index.html) with a status code such as 200 OK, meaning the request succeeded.
7. Browser renders the content and shows it to the user.
/etc/hosts tip: you can manually add DNS records by writing the IP followed by the domain name. Very useful in labs where a target hostname has no public DNS.
4. cURL Basics
• cURL (client URL) is a command-line tool and library that primarily supports HTTP, plus many other protocols.
• Great for scripting and automation.
• Unlike a browser, it does not render HTML/JS/CSS. It prints the raw output.
• Pentesters care mostly about the request and response context, and cURL makes that faster and more convenient than a browser.
• The two core tools for web pentesting in this module: a web browser and cURL.
Basic request
curl http://info.cern.ch/
Prints the raw HTML of the page. (info.cern.ch is the restored first website ever made, created by Tim Berners-Lee at CERN.)
Saving output to a file
curl -O http://info.cern.ch/index.html      # saves using the remote file name
curl -o myfile.html http://info.cern.ch/    # saves using a name you choose
curl -s -O http://info.cern.ch/index.html   # silent: no progress/status output
• -O (capital) keeps the remote filename.
• -o (lowercase) lets you set the filename.
• -s hides the progress/status table.
5. cURL Flag Cheat Sheet
Flag
Long form
Purpose
-d
--data <data>
Send HTTP POST data
-h
--help <category>
Show help (e.g. -h http)
-i
--include
Include response headers in the output
-o
--output <file>
Write output to a named file
-O
--remote-name
Write output to a file named as the remote file
-s
--silent
Silent mode
-u
--user <user:password>
Send server credentials
-A
--user-agent <name>
Set the User-Agent header
-v
--verbose
Show more detail about the request/response
Getting more help:
• curl -h shows a stripped-down menu split into categories.
• curl --help all shows the full option list.
• curl --help <category> (e.g. -h http) shows detailed help for one category.
• man curl opens the full manual page.
6. Quick Review (Self-Test)
1. What are the only two mandatory parts of a URL? (scheme and host)
2. What are the default ports for HTTP and HTTPS? (80 and 443)
3. Where does the browser look before querying a DNS server? (/etc/hosts)
4. What does a 200 OK status code mean? (the request was processed successfully)
5. What is the difference between -o and -O? (-o = your filename, -O = remote filename)
6. Which flag hides cURL's progress output? (-s)
7. Which flag shows response headers? (-i)
8. Which flag gives extra detail on the request and response? (-v)
7. Key Takeaways
• HTTP is a client/server request-response protocol; the browser and cURL both just send requests and read responses.
• Knowing URL structure matters because credentials, parameters, and paths all show up in it, and these are things you will inspect and manipulate while testing.
• The flow is: /etc/hosts check, then DNS, then GET request, then response with a status code, then render.
• cURL shows raw responses, which is what you want for analysis and scripting.
• Flags covered here (-d, -i, -u, -A, -v) will be used in upcoming sections, so make sure you remember what each does.
