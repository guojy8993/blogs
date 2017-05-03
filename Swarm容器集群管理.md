### 本文档内容提要 ###
1. Swarm集群的创建

2. Swarm集群的可用性验证

3. 配置Swarm集群管理可视化WebUI

_ _ _
#### 第一部分: Swarm集群的创建 ####
测试环境说明:
```
节点名称    集群角色    节点IP             节点软件环境      节点必需服务
manager01  master     192.168.232.144   docker          swarm-master
consul0    consul     192.168.232.143   docker          consul
worker01   agent      192.168.232.145   docker          swarm-agent
worker02   agent      192.168.232.141   docker          swarm-agent
client     -          192.168.232.146   docker          -
```

各节点通用配置说明:
```
(1) 更新系统: yum update -y
(2) 安装docker并设置DOCKER_OPTIONS
    [root@worker01 ~]# cat /etc/sysconfig/docker | grep OPTIONS   # 以worker01为例
    OPTIONS='-H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock \
             --selinux-enabled --log-driver=journald --signature-verification=false'
    # 注意: 如果系统TLS Enabled，那么其相关配置亦附加于此处(生产环境下是必须的 TODO)
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
```

#### 第二部分: Swarm集群的可用性验证 ####
检测集群整体信息:
```
```

测试:使用RemoteAPI调用的方式创建容器:
```
```

测试:使用RemoteAPI调用的方式查询各个节点容器分布:
```
```

#### 第三部分: 配置Swarm集群管理可视化WebUI ####
