Interacting with Common Services — Notes

Quick-reference for enumerating and interacting with the services covered: SMB/file shares, Email, and Databases (MySQL/MSSQL), plus tools and troubleshooting.
1. File Share Services

Internal: SMB, NFS, FTP, TFTP, SFTP. External/cloud: Dropbox, OneDrive, SharePoint, S3, Azure Blob, GCS.
SMB — Windows

GUI: WINKEY + R → \\<IP>\<Share>\

CMD

dir \\192.168.220.129\Finance\                      REM list share contents
net use n: \\192.168.220.129\Finance                REM map share to drive N:
net use n: \\192.168.220.129\Finance /user:plaintext Password123   REM with creds
dir n: /a-d /s /b | find /c ":\"                     REM count all files recursively
dir n:\*cred* /s /b                                  REM find files by name pattern
findstr /s /i cred n:\*.*                            REM search file contents for string

    /a-d = files not dirs, /s = recurse, /b = bare output (no headers).

PowerShell

Get-ChildItem \\192.168.220.129\Finance\             # list share (alias: gci / dir)
New-PSDrive -Name "N" -Root "\\192.168.220.129\Finance" -PSProvider "FileSystem"

# With credentials — build a PSCredential object first
$username = 'plaintext'
$password = 'Password123'
$secpassword = ConvertTo-SecureString $password -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential $username, $secpassword
New-PSDrive -Name "N" -Root "\\192.168.220.129\Finance" -PSProvider "FileSystem" -Credential $cred

N:
(Get-ChildItem -File -Recurse | Measure-Object).Count       # count files
Get-ChildItem -Recurse -Path N:\ -Include *cred* -File      # find by filename
Get-ChildItem -Recurse -Path N:\ | Select-String "cred" -List   # search contents (grep-like)

SMB — Linux

sudo mkdir /mnt/Finance
sudo mount -t cifs -o username=plaintext,password=Password123,domain=. //192.168.220.129/Finance /mnt/Finance

# Or with a credentials file (avoids creds in shell history)
mount -t cifs //192.168.220.129/Finance /mnt/Finance -o credentials=/path/credentialfile

credentialfile format:

username=plaintext
password=Password123
domain=.

    Needs cifs-utils: sudo apt install cifs-utils

Once mounted, treat it like any local directory:

find /mnt/Finance/ -name *cred*          # find by filename
grep -rn /mnt/Finance/ -ie cred          # search contents recursively, case-insensitive

Good search terms for creds/sensitive files: cred, password, users, secrets, key Source-code extensions worth grepping: .cs .c .go .java .php .asp .aspx .html

Other file services: FTP, TFTP, NFS — mount/connect with the appropriate tool, then reuse the same local find/grep workflow.
2. Email

Two protocols needed: SMTP (send) + POP3/IMAP (receive).

    Client example: Evolution (GNOME) — sudo apt-get install evolution
        Sandbox error fix: export WEBKIT_FORCE_SANDBOX=0 && evolution
    Connect using the mail server's domain/IP; if SMTPS/IMAPS, need the right encryption (TLS on dedicated port, or STARTTLS). Use "Check for Supported Types" to confirm auth method.

3. Databases

Relational DBs of interest: MySQL and MSSQL. Three interaction methods:

    CLI utilities (mysql, sqsh, sqlcmd)
    Programming languages
    GUI apps (HeidiSQL, MySQL Workbench, SSMS, dbeaver)

MSSQL

# Linux
sqsh -S 10.129.20.13 -U username -P Password123

:: Windows
sqlcmd -S 10.129.20.13 -U username -P Password123

MySQL

# Linux
mysql -u username -pPassword123 -h 10.129.20.13

:: Windows
mysql.exe -u username -pPassword123 -h 10.129.20.13

    Note: no space between -p and the password.

GUI — dbeaver (cross-platform, supports MSSQL/MySQL/PostgreSQL/etc.)

sudo dpkg -i dbeaver-<version>.deb
dbeaver &

Connect with: target IP, port, DB engine, and credentials.
4. Tools Cheat-Sheet by Service
SMB 	FTP 	Email 	Databases
smbclient 	ftp 	Thunderbird 	mssql-cli
CrackMapExec 	lftp 	Claws 	mycli
SMBMap 	ncftp 	Geary 	mssqlclient.py
Impacket 	filezilla 	MailSpring 	dbeaver
psexec.py 	crossftp 	mutt 	MySQL Workbench
smbexec.py 		mailutils 	SSMS
		sendEmail / swaks / sendmail 	
5. Troubleshooting Access Issues

If a connection to a service fails, check:

    Authentication — wrong/missing creds
    Privileges — authenticated but not authorized
    Network connectivity — reachability to the host
    Firewall rules — port blocked
    Protocol support — service/version mismatch (e.g., SMB version, TLS requirement)

Use error codes/messages to search official docs or community forums for the specific service.
Key Takeaway

The core workflow is repeatable across services: mount/connect → enumerate structure → search for sensitive keywords (filenames + file contents) → escalate access. The same find/grep (Linux) or dir/findstr/Get-ChildItem/Select-String (Windows) pattern applies regardless of which file-sharing service is in play.