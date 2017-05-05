### 本文档内容提要 ###
1. Swarm集群的创建

2. Swarm集群的可用性验证

3. 配置Swarm集群管理可视化WebUI

4. 附录

_ _ _
#### 第一部分: Swarm集群的创建 ####
测试环境说明:
```
节点名称    集群角色        节点IP             节点软件环境      节点必需服务
manager01  master/webui   192.168.232.144   docker          swarm-master&portainer
consul0    consul         192.168.232.143   docker          consul
worker01   agent          192.168.232.145   docker          swarm-agent
worker02   agent          192.168.232.141   docker          swarm-agent
client     -              192.168.232.146   docker          -
ca-server  -              192.168.232.152   -               -
```

各节点通用配置说明:
```
(1) 更新系统: yum update -y
(2) 安装docker并设置DOCKER_OPTIONS
    [root@worker01 ~]# cat /etc/sysconfig/docker | grep OPTIONS   # 以worker01为例
    OPTIONS='-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock'
    
    # 注意: 如果系统TLS Enabled，那么其相关配置如下(ca/cert/key具体路径视实际情况而定):
    [root@manager01 ~]# cat /etc/sysconfig/docker | grep OPTIONS
    OPTIONS='-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock \
             --tlsverify --tlscacert=/etc/docker/.certs/ca.pem --tlscert=/etc/docker/.certs/cert.pem \
             --tlskey=/etc/docker/.certs/key.pem'
    # 那么在TLS Enabled下哪些节点需要配置docker engine的TLS呢? 以及TLS ca/cert/key 如何获取呢?
    问题1: 在swarm集群中swarm node(s)的docker engine需要配置TLS以及swarm manager以及客户端需要,其他诸如服务
           发现，swarm agent是不需要的。
    问题2: TLS ca/cert/key的生成参考下文
    [root@ca-server cas]# openssl genrsa -out cas/ca-priv-key.pem 2048
    [root@ca-server cas]# openssl req -config /etc/pki/tls/openssl.cnf -new -key ca-priv-key.pem \
                                      -x509 -days 1825 -out ca.pem
    [root@ca-server cas]# for node in {manager01,worker01,worker02,client};\
    do \
    openssl genrsa -out ${node}-priv-key.pem 2048; \
    openssl req -subj "/CN=${node}" -new -key ${node}-priv-key.pem -out ${node}.csr; \
    openssl x509 -req -days 1825 -in ${node}.csr -CA ca.pem -CAkey ca-priv-key.pem -CAcreateserial \
                                 -out ${node}-cert.pem -extensions v3_req -extfile /etc/pki/tls/openssl.cnf ; \
    openssl rsa -in ${node}-priv-key.pem -out ${node}-priv-key.pem; \
    ssh ${node} 'mkdir -p /etc/docker/.certs/';
    scp ca.pem root@${node}:/etc/docker/.certs/ca.pem; \
    scp ${node}-cert.pem root@${node}:/etc/docker/.certs/cert.pem; \
    scp ${node}-priv-key.pem root@${node}:/etc/docker/.certs/key.pem; \
    done
    # 注意: ca-server在上述示范中是可以域名解析各个节点的
```
```
(3) 关闭SELINUX: 
    [root@worker01 ~]# sed -i "s/.*SELINUX=.*/SELINUX=disabled/g" /etc/selinux/config # 以worker01为例
    [root@worker01 ~]# setenforce 0
(4) 启动docker服务并设置开机自启
    [root@worker01 ~]# systemctl start docker && systemctl enable docker  # 以worker01为例
(5) 根据各个节点的角色预pull或load必需docker镜像
    [root@worker01 ~]# docker pull docker.io/swarm           # 以worker01为例
    [root@worker01 ~]# docker pull docker.io/alpine          # 以worker01为例
    [root@consul0 ~]# docker pull docker.io/progrium/consul  # 以consul0为例
    [root@manager01 ~]# docker pull docker.io/swarm          # 以manager01为例
    [root@manager01 ~]# docker pull docker.io/portainer/portainer # 以manager01为例
```

服务发现节点配置说明:
```
(1) 参考各节点通用配置说明
```
```
(2) 启动consul"服务发现"服务:
[root@consul0 ~]# docker run -d -p 8500:8500 \
                      --name=consul docker.io/progrium/consul -server -bootstrap -advertise=192.168.232.143
```

master节点配置说明:
```
(1) 参考各节点通用配置说明
```
```
(2) 启动swarm master服务:
[root@manager01 ~]# docker run -d -p 4000:4000 docker.io/swarm manage -H :4000 \
                               --replication --advertise 192.168.232.144:4000 consul://192.168.232.143:8500

# 注意: 在TLS Enabled模式下,swarm manager需要能够按域名解析，且需要TLS ca/cert/key
[root@manager01 ~]# cat /etc/hosts | grep -v localhost
192.168.232.144 manager01
192.168.232.146 client
192.168.232.145 worker01
192.168.232.141 worker02
[root@manager01 ~]# docker run -d -p 4000:4000 -v /etc/hosts:/etc/hosts:ro \
                               -v /etc/docker/.certs/:/certs:ro docker.io/swarm manage \
                               --tlsverify --tlscacert=/certs/ca.pem --tlscert=/certs/cert.pem \
                               --tlskey=/certs/key.pem -H :4000 --replication \
                               --advertise 192.168.232.144:4000 consul://192.168.232.143:8500

```
agent节点配置说明:
```
(1) 参考各节点通用配置说明
```
```
(2) 启动swarm agent服务:
[root@worker01 ~]# docker run -d docker.io/swarm join \
                          --advertise=192.168.232.145:2375 consul://192.168.232.143:8500
[root@worker02 ~]# docker run -d docker.io/swarm join \
                          --advertise=192.168.232.141:2375 consul://192.168.232.143:8500

# 注意: 在TLS Enabled模式下,swarm agent需要按域名注册服务
[root@worker01 ~]# docker run -d docker.io/swarm join --advertise=worker01:2375 consul://192.168.232.143:8500
[root@worker02 ~]# docker run -d docker.io/swarm join --advertise=worker02:2375 consul://192.168.232.143:8500
```

#### 第二部分: Swarm集群的可用性验证 ####

检测集群整体信息:
```
[root@client ~]# docker -H tcp://192.168.232.144:4000 info
...
Nodes: 2
 (unknown): 192.168.232.145:2375
  └ ID: 
  └ Status: Pending
  └ Containers: 0
  └ Reserved CPUs: 0 / 0
  └ Reserved Memory: 0 B / 0 B
  └ Labels: 
  └ UpdatedAt: 2017-05-03T09:40:36Z
  └ ServerVersion: 
 (unknown): 192.168.232.141:2375
  └ ID: 
  └ Status: Pending
  └ Containers: 0
  └ Reserved CPUs: 0 / 0
  └ Reserved Memory: 0 B / 0 B
  └ Labels: 
  └ UpdatedAt: 2017-05-03T09:40:47Z
  └ ServerVersion:
...
```
异常分析:
```
    正常情况下, swarm master通过consul发现其他slave节点，并根据slave节点注册的服务地址进行
节点信息收集。而此时hostname无法取得，只显示了consule提供的salve服务ip。判断是slave节点对
swarm master拒绝访问。
```
验证猜测:
```
# 在worker01上抓包并过滤 swarm master的ip
[root@worker01 ~]# tcpdump -i ens33 | grep 192.168.232.144
...
05:48:24.435698 IP worker01 > 192.168.232.144: ICMP host worker01 unreachable \
- admin prohibited, length 68
...
```
解决办法: 
```
# 删除slave节点上的禁ping规则
[root@worker01 ~]# iptables -D INPUT -j REJECT --reject-with icmp-host-prohibited
# worker02 做同样设置
```
再次查询集群状态:

```
[root@client ~]# docker -H tcp://192.168.232.144:4000 info
...
Nodes: 2
 worker01: 192.168.232.145:2375
  └ ID: KXSL:5SWY:3X7T:UPPL:SWYZ:5OYP:FXNG:GQZM:KBMS:MXJN:OTOU:OHJQ
  └ Status: Healthy
  └ Containers: 1 (1 Running, 0 Paused, 0 Stopped)
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.87 GiB
  └ Labels: kernelversion=3.10.0-514.10.2.el7.x86_64, operatingsystem=CentOS Linux 7 (Core), \
            storagedriver=devicemapper
  └ UpdatedAt: 2017-05-03T10:00:15Z
  └ ServerVersion: 1.12.6
 worker02: 192.168.232.141:2375
  └ ID: GH5H:3X6M:LXPH:C7HP:XLZT:QYXC:4CTE:6PAV:E6HU:JCU6:2HMC:IFQT
  └ Status: Healthy
  └ Containers: 1 (1 Running, 0 Paused, 0 Stopped)
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.87 GiB
  └ Labels: kernelversion=3.10.0-514.10.2.el7.x86_64, operatingsystem=CentOS Linux 7 (Core), \
            storagedriver=devicemapper
  └ UpdatedAt: 2017-05-03T10:00:15Z
  └ ServerVersion: 1.12.6
...
```

测试:使用RemoteAPI调用的方式创建容器:
```
[root@client ~]# docker -H tcp://192.168.232.144:4000 run \
                           -idt --name=alpine01 docker.io/alpine ping 127.0.0.1
931d9f4cde52b22caf265aa6e8dd7551a6fb5b2da0d2f446c5646461a506c656
[root@client ~]# docker -H tcp://192.168.232.144:4000 run \
                           -idt --name=alpine02 docker.io/alpine ping 127.0.0.1
4281efef0fbddb2cbd7368a07427396dc1a21bf4d8ccb745b7b965c2426b910b
```

测试:使用RemoteAPI调用的方式查询各个节点容器分布:
```
[root@client ~]# docker -H tcp://192.168.232.144:4000 ps -a
CONTAINER ID   IMAGE     COMMAND      CREATED       STATUS       PORTS       NAMES
4281efef0fbd   docker.io/alpine "ping 127.0.0.1"  8 seconds ago  Up 7 seconds  worker02/alpine02
931d9f4cde52   docker.io/alpine "ping 127.0.0.1"  14 seconds ago  Up 12 seconds worker01/alpine01
3ceb29f40247   docker.io/swarm  "/swarm join --advert"  24 minutes ago Up 24 minutes 2375/tcp 
worker02/boring_wescoff
33c9e0a49c1b   docker.io/swarm  "/swarm join --advert"  25 minutes ago Up 25 minutes 2375/tcp 
worker01/gloomy_mestorf
```
特别注意1: 在TLS Enabled模式下,调用Docker Remote API需要指定客户端本地保存的ca/cert/key，且需要按域名访问
```
[root@client ~]# docker -H manager01:4000 ps -a
Get http://manager01:4000/v1.24/containers/json?all=1: malformed HTTP response "\x15\x03\x01\x00\x02\x02".
* Are you trying to connect to a TLS-enabled daemon without TLS?

[root@client ~]# docker --tlsverify --tlscacert=/etc/docker/.certs/ca.pem --tlscert=/etc/docker/.certs/cert.pem --tlskey=/etc/docker/.certs/key.pem -H 192.168.232.144:4000 ps -a
An error occurred trying to connect: Get https://192.168.232.144:4000/v1.24/containers/json?all=1: x509: cannot validate certificate for 192.168.232.144 because it doesn't contain any IP SANs

[root@client ~]# docker --tlsverify --tlscacert=/etc/docker/.certs/ca.pem \
                        --tlscert=/etc/docker/.certs/cert.pem --tlskey=/etc/docker/.certs/key.pem \
                        -H manager01:4000 ps -a
CONTAINER ID  IMAGE COMMAND CREATED  STATUS PORTS NAMES
6da1cf875360  docker.io/alpine "ping 127.0.0.1"  About an hour ago   Up About an hour  worker03/alpine03
...
```
特别注意2: 在TLS Enabled模式下,调用Docker Remote API可能碰到错误信息提示: ca expired或invalid, 原因是当前
持有ca的服务器的本地时间在 ca.pem 包含的有效时间区间之外，所以ca-server与其他节点要使用时间服务器进行时间同步。
现举例说明:
```
[root@client ~]# date
Sat May  6 04:46:21 EDT 2017

[root@client ~]# openssl x509 -in /etc/docker/.certs/ca.pem -noout -text | egrep "Validity|Not Before|Not After"
Validity
    Not Before: May  5 14:45:49 2017 GMT
    Not After : May  4 14:45:49 2022 GMT
```
读者遇到上述错误可以使用 date 命令与 openssl x509 -in <key-name> -noout -text 定位该问题

#### 第三部分: 配置Swarm集群管理可视化WebUI ####

在swarm master节点启动portainer容器:
```
[root@manager01 ~]# docker run -idt --name=portainer -p 80:9000 docker.io/portainer/portainer
[root@manager01 ~]# ss -antulp | grep 80
tcp LISTEN 0 128 :::80 :::*  users:(("docker-proxy-cu",pid=5509,fd=4))
```
使用浏览器访问 http://swarm-master:80, 设置admin用户密码，并登录

[设置admin密码](http://pan.baidu.com/s/1cpfI2q)

[登录](http://pan.baidu.com/s/1pL0k9a7)

设置swarm api的endpoint即 swarm-master:4000(注意swarm api依据读者自己实际情况填写)

[设置swarm api endpoint](http://pan.baidu.com/s/1jHW89CU)

```
#NOTE: 在实际情况中我们发现，填写endpoint之后，页面提示错误信息:无法访问指定的链接；
       错误原因跟之前集群pending一样，是部署portainer的节点禁ping了。解决办法同前文。
```
跳转进入管理平台

[使用管理平台管理swarm集群](http://pan.baidu.com/s/1jH4Om02)


#### 附录 ####

[1. Docker Swarm官方安装手册](https://github.com/docker/docker.github.io/blob/master/swarm/install-manual.md)

[2. Portainer官方安装手册](http://portainer.io/install.html)

[3. Docker Swarm配置TLS](https://github.com/docker/docker.github.io/blob/master/swarm/configure-tls.md)


