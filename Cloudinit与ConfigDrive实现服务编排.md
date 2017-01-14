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
服务器名称           角色              IP                   系统       硬件配置       密码
test2.cloud.org      测试客户端        192.168.100.25/24    CentOS7    C1M1024       test2
haproxy.cloud.org    负载均衡服务器    192.168.100.26/24    CentOS7    C4M4096       haproxy
web01.cloud.org      后端web服务器     192.168.100.27/24    CentOS7    C2M2048       web01
web02.cloud.org      后端web服务器     192.168.100.28/24    CentOS7    C2M2048       web02
```

> **NOTE:**

> CentOS7系统镜像是特殊定制版本,安装有CloudInit

> 网络: 方便测试需要，采用简单Flat网络，均连接到某linux网桥(br0)

___
#### 各个节点的配置说明 ####

(1) test2.cloud.org: 需要修改hosts，密码,机器名,设置网络，其他无特殊设置

(2) haproxy.cloud.org: 需要修改hosts，设置网络，密码,机器名,安装haproxy，拷贝 haproxy.cfg，设置服务，其他无特殊设置

(3) 各个web节点: 需要修改hosts，设置网络，密码,机器名,安装httpd，拷贝配置文件，设置服务，其他无特殊设置

> **NOTE:**

> hosts文件需要保持一致

> 测试需要，web节点站点文件略微不同



___
#### 各个节点ConfigDrive的配置与生成 ####
(1) 通用文件的创建以及安装包的准备

i. 创建公用hosts文件
```
[root@dev ~]# cat > /tmp/hosts << EOF
> 127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
> ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
> 192.168.100.25 test2.cloud.org
> 192.168.100.26 haproxy.cloud.org
> 192.168.100.27 web01.cloud.org
> 192.168.100.28 web02.cloud.org
> EOF
```
ii.准备haproxy, httpd的rpm安装包(yum downloadonly或者预先上传,此处不细说,略)
```
[root@dev ~]# ls /tmp/ | egrep "rpm|tar.gz|hosts"
haproxy.rpm
hosts
httpd.tar.gz
```
> **NOTE:**

> dev机器是未来作为编排引擎服务所在的机器,后续生成ConfigDrive的工作都在该服务器上做

> httpd.tar.gz是httpd.rpm以及依赖所rpms的集合

(2) test2.cloud.org 的ConfigDrive的准备

i. 生成config drive最简目录,拷贝 hosts ,制作网卡配置文件(ifcfg-eth0)并拷贝
```
[root@dev test]# cp /tmp/hosts openstack/content/0000
[root@dev test]# cat > openstack/content/0001 << EOF
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
NAME=eth0
ONBOOT=yes
IPADDR0=192.168.100.25
PREFIX0=24
GATEWAY0=192.168.100.1
EOF

[root@dev test]# tree
.
└── openstack
    ├── content
    │   ├── 0000
    │   └── 0001
    └── latest
        ├── meta_data.json
        ├── user_data
        └── vendor_data.json
```
ii.生成meta_data.json
```
{ 
    "files": [ {"path": "/etc/hosts", "content_path": "/content/0000"},
               {"path": "/etc/sysconfig/network-scripts/ifcfg-eth0", "content_path": "/content/0001"}
             ], 
    "hostname": "test2.cloud.org", 
    "launch_index": 0, 
    "name": "test", 
    "uuid": "9478337b-b970-4cfe-ad07-53c8b99c831d"
}
```
> **NOTE:**

> 拷贝 hosts 以及 网卡配置文件,设置机器名

iii.生成user_data脚本
```
[root@dev test]# cat > openstack/latest/user_data << EOF
#!/bin/bash
echo test2 | passwd --stdin root
systemctl restart network
```

iv.打包iso镜像
```
[root@dev test]# mkisofs -R -V config-2 -o /root/test2.iso ./
```

(3) haproxy.cloud.org 的ConfigDrive的准备
i. 生成config drive最简目录,拷贝 hosts ,制作网卡配置文件(ifcfg-eth0)并拷贝,拷贝haproxy.rpm及其配置文件
```
[root@dev haproxy]# cp /tmp/hosts openstack/content/0000
[root@dev haproxy]# cat > openstack/content/0001 << EOF
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
NAME=eth0
ONBOOT=yes
IPADDR0=192.168.100.26
PREFIX0=24
GATEWAY0=192.168.100.1
EOF

[root@dev haproxy]# cp /tmp/haproxy.rpm openstack/content/0002
[root@dev haproxy]# cat openstack/content/0003
global
   daemon
   user nobody
   group haproxy
   log /dev/log local0
   log /dev/log local1 notice
   stats socket /tmp/sock
   mode 0666 level user

defaults
   log global
   retries 3
   option redispatch 
   timeout connect 5000
   timeout client 50000
   timeout server 50000
frontend c4c49fc1-bf10-4270-bc20-02b93439897f
   option tcplog
   bind haproxy.cloud.org
   mode http
   default_backend 8f09dbaf-0c78-48f1-8b80-662e75036485
   maxconn 100
   option forwardfor
backend 8f09dbaf-0c78-48f1-8b80-662e75036485
   mode http
   balance leastconn
   option forwardfor
   server 0c7d3741-df7b-4bf3-9396-14cc3c05b5a5 web01.cloud.org weight 100
   server 44c18162-301e-42f3-9f0a-5522f426412f web02.cloud.org weight 100
```

ii.生成meta_data.json
```
{
    "files": [ {"path": "/etc/hosts", "content_path": "/content/0000"},
               {"path": "/etc/sysconfig/network-scripts/ifcfg-eth0", "content_path": "/content/0001"},
               {"path": "/tmp/haproxy.rpm", "content_path": "/content/0002"},
               {"path": "/tmp/haproxy.cfg", "content_path": "/content/0003"}
             ], 
    "hostname": "haproxy.cloud.org", 
    "launch_index": 0, 
    "name": "haproxy", 
    "uuid": "302d0f16-a8ff-4138-a80f-c95a4ff5787e"
}
```
iii.生成user_data脚本
```
[root@dev haproxy]# cat openstack/latest/user_data 
#!/bin/bash
echo haproxy | passwd --stdin root
systemctl restart network
rpm -ivh /tmp/haproxy.rpm && rm -rf /tmp/haproxy.rpm
haproxy -f /tmp/haproxy.cfg
```
iv.打包iso镜像
```
[root@dev haproxy]# mkisofs -R -V config-2 -o /root/haproxy.iso ./
```

(4) web01.cloud.org 的ConfigDrive的准备
i. 生成config drive最简目录,拷贝 hosts ,制作网卡配置文件(ifcfg-eth0)并拷贝
```
[root@dev web01]# cp /tmp/hosts openstack/content/0000
[root@dev web01]# cat > openstack/content/0001 << EOF
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
NAME=eth0
ONBOOT=yes
IPADDR0=192.168.100.27
PREFIX0=24
GATEWAY0=192.168.100.1
EOF

[root@dev web01]# cp /tmp/httpd.tar.gz openstack/content/0002
[root@dev web01]# cat > openstack/content/0003 << EOF
Greetings from web01.cloud.org !
EOF
```

ii.生成meta_data.json
```
{
    "files": [ {"path": "/etc/hosts", "content_path": "/content/0000"},
               {"path": "/etc/sysconfig/network-scripts/ifcfg-eth0", "content_path": "/content/0001"},
               {"path": "/tmp/httpd.tar.gz", "content_path": "/content/0002"},
               {"path": "/tmp/index.html", "content_path": "/content/0003"}
             ], 
    "hostname": "web01.cloud.org", 
    "launch_index": 0, 
    "name": "web01", 
    "uuid": "282101d0-4eea-4430-a928-5a7976283898"
}
```

iii.生成user_data脚本
```
```
iv.打包iso镜像

(5) web02.cloud.org 的ConfigDrive的准备
i. 生成config drive最简目录,拷贝 hosts ,制作网卡配置文件(ifcfg-eth0)并拷贝

ii.生成meta_data.json

iii.生成user_data脚本

iv.打包iso镜像


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
