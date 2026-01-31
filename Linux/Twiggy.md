##### Tags: `Writable file`  `SUID-l`  `Python Flask`  `Web-enum`

# 🐧Twiggy🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.242.62 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-31 18:38 AEDT
Nmap scan report for 192.168.242.62
Host is up (0.22s latency).
Not shown: 65529 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
53/tcp   open  domain  NLnet Labs NSD
80/tcp   open  http    nginx 1.16.1
4505/tcp open  zmtp    ZeroMQ ZMTP 2.0
4506/tcp open  zmtp    ZeroMQ ZMTP 2.0
8000/tcp open  http    nginx 1.16.1

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 406.01 seconds
```
check the web and didn't have anything  
then search ztmp exploit got the RCE script
```
(https://github.com/user-attachments/assets/66675ad8-9f5a-4e91-a08e-121dbfd70979)
```
