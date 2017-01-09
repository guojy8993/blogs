### 本文档章节目录 ###
```
(1) Provider网络的初始化
(2) 租户网络的初始化
(3) 各个计算节点上租户容器与虚拟机的创建
(4) 各节点bridge添加fdb entry实现跨宿主租户内网连通
(5) 为租户内网IP绑定浮动IP
(6) 负载均衡即服务实现
(7) 附录
```

![kvm_vxlan_docker混合云](https://github.com/guojy8993/blogs/blob/master/sys.png)

#### 第一节:Provider网络的初始化 ####


#### 第二节:租户网络的初始化 ####
#### 第三节:各个计算节点上租户容器与虚拟机的创建 ####
#### 第四节:各节点bridge添加fdb entry实现跨宿主租户内网连通 ####

为了保证各个租户业务以及网关网络互通,需要从以下3方面进行配置:

(1)为网络节点的租户网络网桥添加fdb entry,处理arp包(目标mac为00:00:00:00:00:00,通过vtep设备forward到各个计算节点租户vtep)

(2)为网络节点的租户网络网桥添加fdb entry,处理单播包(目标明确,通过vtep设备forward到指定计算节点租户vtep)

(3) 为各个计算节点的租户网络网桥添加fdb entry,处理arp包(目标mac为00:00:00:00:00:00,通过vtep设备forward到网络节点以及除自己外的租户业务宿主vtep)

fdb添加entry的语法格式(具体参数说明请 man bridge fdb):
```
bridge fdb {add|append|del|replace} LLADDR dev DEV {local|temp} {self} {router} 
                                    [dst IPADDR] [vni VNI] [port PORT] [via DEVICE]
```

此处举例说明(此处仅举例说明,并非详细配置):

(1)网络节点(10.160.0.127) - 从网络节点租户私有网络网关(初次)访问租户业务(192.168.100.6),MAC未知的情况:广播到该租户业务分布的业务宿主(10.160.0.126,10.160.0.154)
```
bridge fdb add 00:00:00:00:00:00 dev vxlan-100 dst 10.160.0.126
bridge fdb add 00:00:00:00:00:00 dev vxlan-100 dst 10.160.0.154
```

(2)网络节点(10.160.0.127) - 从网络节点租户私有网络网关访问租户业务(192.168.100.6),MAC已知的情况:单播到该租户业务所在宿主(10.160.0.126)
```
bridge fdb replace 52:54:00:7c:d6:2c dev vxlan-66 dst 10.160.0.126
```
(3)计算节点(10.160.0.126) - 从业务(192.168.100.6)访问同租户私有网络其他设备(192.168.100.1),MAC未知的情况:广播到该租户业务分布的业务宿主(当前宿主除外,10.160.0.154)以及网络节点(10.160.0.127)
```
bridge fdb add 00:00:00:00:00:00 dev vxlan-100 dst 10.160.0.127
bridge fdb add 00:00:00:00:00:00 dev vxlan-100 dst 10.160.0.154
```

(4)计算节点(10.160.0.154) - 从某业务(e.g:web)访问同租户其他业务或设备(e.g:db),MAC已知的情况:单播到目标业务所在宿主
```
bridge fdb replace 0e:49:e8:8a:9e:e7 dev vxlan-100 dst 10.160.0.126
```

考虑到:

(1)添加fdb entry工作的浩繁工作量

(2)人工操作的不可靠性

(3)依靠中控程序下刷并实时更新fdb entry可能引发单点故障

由此设想: 各个计算节点拥有自己的agent程序,职责如下:

(1)作为可能的mesos slave的executor程序,负责具体业务及其附属网络设施的创建

(2)维护基础租户网络 vxlan-XX/bridge设备

(3)在新计算节点加入集群并创建某租户业务时,通过消息队列向同租户业务所在各个宿主宣告自己的诞生

(4)负责监听各租户新加入业务的消息,更新fdb entry,保证新业务所在私有网络各个节点互通

(5)在某租户在某宿主上的最后一个业务删除时,通过消息队列向同租户其他业务宣告自己的终结,并清除租户vxlan-XX/bridge设备

(6)负责监听各个租户删除业务的消息,更新fdb entry,保证之后租户内arp不会向已删除业务所在的宿主进行广播

(7)对外提供API:业务生命周期(e.g:业务重启后的网络重建)管理,镜像管理,存储管理这样就可以实现多个agent之间的实时/异步互动,而中控程序主要负责业务创建的调用以及业务信息保存查询

总之,集群可以实现自管理,更易于人员维护;且去中心化,横向扩展能力更好

#### 第五节:为租户内网IP绑定浮动IP ####

(1) 添加租户内网ip与浮动ip的nat映射(以192.168.100.6绑定浮动ip200.160.0.4为例)
```
[root@docker-net127 ~]# ip netns exec private-router \
ip addr add 200.160.0.4/32 broadcast 200.160.0.4  dev ss-router-gw
[root@docker-net127 ~]# ip netns exec private-router \
iptables -t nat -A PREROUTING -d 200.160.0.4 -j DNAT --to-destination 192.168.100.6
[root@docker-net127 ~]# ip netns exec private-router \
iptables -t nat -A OUTPUT -d 200.160.0.4 -j DNAT --to-destination 192.168.100.6
[root@docker-net127 ~]# ip netns exec private-router \
iptables -t nat -A POSTROUTING -s 192.168.100.6 -j SNAT --to-source 200.160.0.4
```
(2)公网测试web服务以及ssh可用性
```
[root@net ~]# curl 200.160.0.4:9898
This Page is got from server 192.168.100.6(nat to 200.160.0.4)
[root@net ~]# ssh root@200.160.0.4
root@200.160.0.4's password:
Last login: Sun Nov 20 16:47:51 2016 from 200.160.0.1
[root@localhost ~]#
```

#### 第六节:负载均衡即服务实现 ####
(1) 添加负载均衡前端ip与设备

```
[root@docker-net127 ~]# ip link add name ha-if type veth peer name ha-out
[root@docker-net127 ~]# ip link set ha-if mtu 1450
[root@docker-net127 ~]# brctl addif private-bridge ha-out
[root@docker-net127 ~]# ip netns add ha-vxlan100-web
[root@docker-net127 ~]# ip link set ha-if netns ha-vxlan100-web
[root@docker-net127 ~]# ip netns exec ha-vxlan100-web ip link set lo up
[root@docker-net127 ~]# ip link set ha-out mtu 1450 up
[root@docker-net127 ~]# ip netns exec ha-vxlan100-web ip addr add dev ha-if 192.168.100.8/24
[root@docker-net127 ~]# ip netns exec ha-vxlan100-web ip route add default via 192.168.100.1 dev ha-if
```

(2) 在vxlan网络(某租户网络)内启动负载均衡服务

```
[root@docker-net127 ~]# mkdir -p /var/lib/netns/ha-vxlan100-web/
[root@docker-net127 ~]# cat > /var/lib/netns/ha-vxlan100-web/haproxy.cfg << EOF
global
   daemon
   user nobody
   group haproxy
   log /dev/log local0
   log /dev/log local1 notice
defaults
   log global
   retries 3
   option redispatch
   timeout connect 5000
   timeout client 50000
   timeout server 50000
frontend c4c49fc1-bf10-4270-bc20-02b93439897f
   option tcplog
   bind 192.168.100.8:80
   mode http
   default_backend 8f09dbaf-0c78-48f1-8b80-662e75036485
   maxconn 100
   option forwardfor
backend 8f09dbaf-0c78-48f1-8b80-662e75036485
   mode http
   balance leastconn
   option forwardfor
   server 0c7d3741-df7b-4bf3-9396-14cc3c05b5a5 192.168.100.6:9898 weight 100
   server 44c18162-301e-42f3-9f0a-5522f426412f 192.168.100.7:9898 weight 100
EOF
```
```
[root@docker-net127 ~]# ip netns exec ha-vxlan100-web haproxy -f /var/lib/netns/ha-vxlan100-web/haproxy.cfg
```

(3) 租户内网测试负载均衡服务

```
[root@docker-net127 ~]# ip netns exec private-router curl 192.168.100.8
This Page is got from server 192.168.100.6(nat to 200.160.0.4)
[root@docker-net127 ~]# ip netns exec private-router curl 192.168.100.8
This page is got from 192.168.100.7
```

(4) 为负载均衡前端ip绑定浮动ip(200.160.0.5),对公网提供服务

```
[root@docker-net127 ~]# ip netns exec private-router \
ip addr add 200.160.0.5/32 broadcast 200.160.0.5  dev ss-router-gw
[root@docker-net127 ~]# ip netns exec private-router \
iptables -t nat -A PREROUTING -d 200.160.0.5 -j DNAT --to-destination 192.168.100.8
[root@docker-net127 ~]# ip netns exec private-router \
iptables -t nat -A OUTPUT -d 200.160.0.5 -j DNAT --to-destination 192.168.100.8
[root@docker-net127 ~]# ip netns exec private-router \
iptables -t nat -A POSTROUTING -s 192.168.100.8 -j SNAT --to-source 200.160.0.5
```

(5) 公网测试负载均衡服务

```
[root@net ~]# curl 200.160.0.5
This Page is got from server 192.168.100.6(nat to 200.160.0.4)
[root@net ~]# curl 200.160.0.5
This page is got from 192.168.100.7
```

#### 附录 ####
[^1^] [KVM vxlan docker 混合云详细图](https://github.com/guojy8993/blogs/blob/master/kvm%E4%B8%8Edocker%E6%B7%B7%E5%90%88%E4%BA%91.png)

[^2^] [计算节点docker容器初始化脚本](https://github.com/guojy8993/blogs/blob/master/Docker%E9%AB%98%E7%BA%A7%E7%BD%91%E7%BB%9C%E9%85%8D%E7%BD%AE%E5%AE%9E%E6%88%98)
