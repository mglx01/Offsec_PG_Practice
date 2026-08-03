Enumeration
$ nmap -p- -T4 -sV 10.129.244.180
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-03 22:41 AEST
Nmap scan report for 10.129.244.180
Host is up (0.013s latency).
Not shown: 65384 filtered tcp ports (no-response), 148 filtered tcp ports (host-prohibited)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
80/tcp   open  http    nginx 1.12.2
9200/tcp open  http    nginx 1.12.2
