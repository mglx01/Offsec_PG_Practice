##### Tags: `cronjob`  `CVE-2021–22204`  `searchsploit`  `User-add`

# 🐧Astronaut🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.156.12 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-01 16:55 AEDT
Nmap scan report for 192.168.156.12
Host is up (0.25s latency).
Not shown: 65511 closed tcp ports (reset)
PORT      STATE    SERVICE     VERSION
22/tcp    open     ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp    open     http        Apache httpd 2.4.41
```
The web is running 
