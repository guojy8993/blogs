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
consul01   consul     192.168.232.143   docker          consul
worker01   agent      192.168.232.145   docker          swarm-agent
worker02   agent      192.168.232.141   docker          swarm-agent
client     -          192.168.232.146   docker          -
```

各节点通用配置说明:
```
(1) 更新系统: yum update -y
(2) 安装docker并设置DOCKER_OPTIONS
```

```
(3) 关闭SELINUX
(4) 启动docker服务并设置开机自启
(5) 根据各个节点的角色预pull或load必需docker镜像
```

master节点配置说明:
```
```

agent节点配置说明:
```
```

服务发现节点配置说明:
```
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
