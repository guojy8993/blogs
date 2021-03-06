### 本文档内容提要
第一部分 Linux下的跨进程通信

第二部分 使用sock实现本地跨进程通信

第三部分 本地跨进程sock通信在Neutron项目中的典型应用(网络节点租户网络穿透)


#### 第一部分 Linux下的跨进程通信 ##

#### 第二部分 使用sock实现本地跨进程通信 ##
```
原理: 多进程均使用AF_UNIX地址族(family),通过连接到同一.sock,实现C/S模式的通信; 
       而需要优先启动某server角色的进程,生成套接字文件,其他进程通过AF_UNIX地址族的
       socket连接于同一socket文件;
       以下是简单的C/S程序示例:
```

1. server程序(server.py)

```
#! /usr/bin/env python
import socket
import os
import json
import threading
buf_size = 1024 * 1024
unix_sock = "/tmp/unix_ipc.sock"

def fake_handler(conn):
    disconnect = False
    while not disconnect:
        try:
            msg = conn.recv(buf_size)
            conn.send("[*] thanks for your request: %s" % msg)
        except Exception:
            disconnect = True
            conn.close()
server = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
if os.path.exists(unix_sock):
    os.unlink(unix_sock)
server.bind(unix_sock)
server.listen(0)
while 1:
    conn, addr = server.accept()
    task = threading.Thread(target=fake_handler,args=(conn,))
    task.start()
```

2.client程序(client.py)
```
#! /usr/bin/env python
import socket
import os
import sys
import json
buf_size = 1024 * 1024
unix_sock = "/tmp/unix_ipc.sock"
client = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
client.connect(unix_sock)
exit_sig_recvd = False

while not exit_sig_recvd:
    print "[*] Input:"
    input = raw_input()
    if "exit" in input or "quit" in input:
        exit_sig_recvd = True
        continue
    try:
        client.send(input)
        print "%s" % client.recv(buf_size)
    except Exception:
        exit_sig_recvd = True
        continue
client.close()
print "[*] exit gracefully !"
```
#### 本地跨进程sock通信在Neutron项目中的典型应用 ##
租户网络穿透的需求场景: 虚拟机实例启动后,模板内置的cloud-init服务会通过租户网络访问 http://169.154.169.254

#### 附录
[Python socket 实现进程间通信 ](http://blog.csdn.net/wangtaoking1/article/details/44494217/)
