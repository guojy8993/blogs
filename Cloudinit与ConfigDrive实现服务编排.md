### CloudInit + ConfigDrive 实现云编排(以简单的负载均衡为例) ###

(1) 云编排测试环境需求

(2) 各个节点的配置说明

(3) 各个节点ConfigDrive的配置与生成

(4) 启动各个节点实现服务编排

(5) 检验服务编排

___
#### 云编排测试环境需求 ####
(1) 在本次测试中我们需要4个服务器，其信息如下所示：
```
服务器名称           角色              IP                   系统       硬件配置
test2.cloud.org      测试客户端        192.168.100.25/24    CentOS7    C1M1024
haproxy.cloud.org    负载均衡服务器    192.168.100.26/24    CentOS7    C4M4096
web01.cloud.org      后端web服务器     192.168.100.27/24    CentOS7    C2M2048
web02.cloud.org      后端web服务器     192.168.100.28/24    CentOS7    C2M2048
```

> **NOTE:**

> CentOS7系统镜像是特殊定制版本,安装有CloudInit

> 网络: 方便测试需要，采用简单Flat网络，均连接到某linux网桥(br0)

___
#### 各个节点的配置说明 ####

(1) test2.cloud.org: 需要修改hosts，设置网络，其他无特殊设置

(2) haproxy.cloud.org: 需要修改hosts，设置网络，安装haproxy，拷贝 haproxy.cfg，设置服务，其他无特殊设置

(3) 各个web节点: 需要修改hosts，设置网络，安装nginx，拷贝配置文件，设置服务，其他无特殊设置

> **NOTE:**

> hosts文件需要保持一致

> 测试需要，web节点站点文件略微不同



___
#### 各个节点ConfigDrive的配置与生成 ####

___
#### 启动各个节点实现服务编排 ####


___
#### 检验服务编排 ####



#### 参考链接 ####
[1. CloudInit镜像制作](https://github.com/guojy8993/blogs/blob/master/OpenStack%E9%95%9C%E5%83%8F%28%E5%9F%BA%E4%BA%8ECentOS7%29%E7%9A%84%E5%88%B6%E4%BD%9C%E4%B8%8E%E8%AF%B4%E6%98%8E)

[2. ConfigDrive文件系统组织与数据格式](https://github.com/guojy8993/blogs/blob/master/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8ConfigDrive%E5%AE%9E%E7%8E%B0%E4%BA%91%E5%AE%9E%E4%BE%8B%E5%88%9D%E5%A7%8B%E5%8C%96.md)

[3. Haproxy负载均衡](http://blog.csdn.net/tantexian/article/details/50056199)

[4. 如何使用mkisofs制作镜像文件](http://blog.csdn.net/taiyang1987912/article/details/42563597)

[5. 如何使用virt-install启动虚拟机](http://www.361way.com/virt-install/2721.html)
