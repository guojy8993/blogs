### 文章章节目录 
1. Provider网络的初始化
2. 租户网络的初始化
3. 各计算节点上租户docker容器与kvm虚拟机的创建
4. 各节点bridge添加fdb entry实现跨宿主租户内网连通
5. 为租户内网IP绑定浮动IP
6. 负载均衡即服务实现
7. 计算节点docker容器网络初始化脚本

![kvm+vxlan+docker混合云](https://github.com/guojy8993/blogs/blob/master/sys.png)
#### 第一节 Provider网络的初始化
(1)创建privider网络的网桥
```
[root@docker-net127 ~]# brctl addbr public-bridge
[root@docker-net127 ~]# brctl stp public-bridge off
[root@docker-net127 ~]# brctl sefd public-bridge 0
[root@docker-net127 ~]# ip link set public-bridge mtu 1500 up
```
(2)连接provider网桥到宿主provider网络网卡(ens192)
```
[root@docker-net127 ~]# brctl addif public-bridge ens192
[root@docker-net127 ~]# ip link set ens192 up
```
(3)创建provider网络的dhcp服务,创建ip资源池
```
[root@docker-net127 ~]# ip link add pub-dhcp-out type veth peer name pub-dhcp-in
[root@docker-net127 ~]# brctl addif public-bridge pub-dhcp-out
[root@docker-net127 ~]# ip netns add public-dhcp
[root@docker-net127 ~]# ip link set pub-dhcp-in netns public-dhcp
[root@docker-net127 ~]# ip netns exec public-dhcp ip link set lo up
[root@docker-net127 ~]# ip netns exec public-dhcp ip link set lo state up
[root@docker-net127 ~]# ip netns exec public-dhcp ip link set pub-dhcp-in mtu 1500 up
[root@docker-net127 ~]# ip link set pub-dhcp-out mtu 1500 up
[root@docker-net127 ~]# ip netns exec public-dhcp ip addr add \
dev pub-dhcp-in 200.160.0.2/24
[root@docker-net127 ~]# ip netns exec public-dhcp ip route add default \
via 200.160.0.1 dev pub-dhcp-in
[root@docker-net127 ~]# ip netns exec public-dhcp ping -c 1 200.160.0.1
[root@docker-net127 ~]# uuidgen
81758417-bf10-4cb0-af2f-5d3a0c76c792
[root@docker-net127 ~]# mkdir -p /tmp/dhcp/81758417-bf10-4cb0-af2f-5d3a0c76c792/
[root@docker-net127 ~]# ip netns exec public-dhcp dnsmasq --no-hosts \
      --no-resolv --strict-order --except-interface=lo \
      --pid-file=/tmp/dhcp/81758417-bf10-4cb0-af2f-5d3a0c76c792/pid \
      --dhcp-hostsfile=/tmp/dhcp/81758417-bf10-4cb0-af2f-5d3a0c76c792/host \
      --addn-hosts=/tmp/dhcp/81758417-bf10-4cb0-af2f-5d3a0c76c792/addn_hosts \
      --dhcp-optsfile=/tmp/dhcp/81758417-bf10-4cb0-af2f-5d3a0c76c792/opts \
      --dhcp-leasefile=/tmp/dhcp/81758417-bf10-4cb0-af2f-5d3a0c76c792/leases \
      --dhcp-match=set:ipxe,175 --bind-interfaces --interface=pub-dhcp-in \
      --dhcp-range=set:tag0,200.160.0.0,static,86400s \
      --dhcp-option-force=option:mtu,1500 \
      --dhcp-lease-max=256 \
      --conf-file= \
      --domain=provider-dhcp
```
备注: Provider网络的网络设备mtu全部1500

#### 第二节 租户网络的初始化
(1)创建租户网络的网桥
```
[root@docker-net127 ~]# brctl addbr private-bridge
[root@docker-net127 ~]# brctl stp private-bridge off
[root@docker-net127 ~]# brctl setfd private-bridge 0
[root@docker-net127 ~]# ip link set private-bridge mtu 1450 up
```

(2)创建租户网络的dhcp服务,创建ip资源池
```
[root@docker-net127 ~]# ip netns add private-dhcp
[root@docker-net127 ~]# ip link add name ss-dhcp-out type veth peer name ss-dhcp-in
[root@docker-net127 ~]# brctl addif private-bridge ss-dhcp-in
[root@docker-net127 ~]# ip link set ss-dhcp-out mtu 1450 up
[root@docker-net127 ~]# ip link set ss-dhcp-in netns private-dhcp
[root@docker-net127 ~]# ip netns exec private-dhcp ip link set lo up
[root@docker-net127 ~]# ip netns exec private-dhcp ip link set lo state up
[root@docker-net127 ~]# ip netns exec private-dhcp ip link set ss-dhcp-in mtu 1450 up
[root@docker-net127 ~]# ip netns exec private-dhcp ip addr add \
dev ss-dhcp-in 192.168.100.2/24
[root@docker-net127 ~]# ip netns exec private-dhcp ip route add default \
via 192.168.100.1 dev ss-dhcp-in
[root@docker-net127 ~]# uuidgen
bd406409-135e-44b1-a4ed-f4d6365118fb
[root@docker-net127 ~]# mkdir -p /tmp/dhcp/bd406409-135e-44b1-a4ed-f4d6365118fb/
[root@docker-net127 ~]# ip netns exec private-dhcp dnsmasq --no-hosts \
       --no-resolv --strict-order --except-interface=lo \
       --pid-file=/tmp/dhcp/bd406409-135e-44b1-a4ed-f4d6365118fb/pid \
       --dhcp-hostsfile=/tmp/bd406409-135e-44b1-a4ed-f4d6365118fb/host \
       --addn-hosts=/tmp/dhcp/bd406409-135e-44b1-a4ed-f4d6365118fb/addn_hosts \
       --dhcp-optsfile=/tmp/dhcp/bd406409-135e-44b1-a4ed-f4d6365118fb/opts \
       --dhcp-leasefile=/tmp/dhcp/bd406409-135e-44b1-a4ed-f4d6365118fb/leases \
       --dhcp-match=set:ipxe,175 --bind-interfaces --interface=ss-dhcp-in \
       --dhcp-range=set:tag0,192.168.100.0,static,86400s \
       --dhcp-option-force=option:mtu,1450 \
       --dhcp-lease-max=256 \
       --conf-file= \
       --domain=ss-dhcp
```

(3)创建租户的私有路由,并连接到租户内网
```
[root@docker-net127 ~]# ip netns add private-router
[root@docker-net127 ~]# ip netns exec private-router ip link set lo up
[root@docker-net127 ~]# ip netns exec private-router ip link set lo state up
[root@docker-net127 ~]# ip link add ss-rt-out type veth peer name ss-rt-in
[root@docker-net127 ~]# ip link set ss-rt-in netns private-router
[root@docker-net127 ~]# ip netns exec private-router ip link set ss-rt-in mtu 1450 up
[root@docker-net127 ~]# ip link set ss-rt-out mtu 1450 up
[root@docker-net127 ~]# brctl addif private-bridge ss-rt-out
[root@docker-net127 ~]# ip netns exec private-router ip addr add \
dev ss-rt-in 192.168.100.1/24
[root@docker-net127 ~]# ip netns exec private-router sysctl -w net.ipv4.ip_forward=1
```
(4)连接私有路由到provider网络(设置路由网关)
```
[root@docker-net127 ~]# ip link add name ss-router-out type veth peer name ss-router-gw
[root@docker-net127 ~]# ip link set ss-router-gw netns private-router
[root@docker-net127 ~]# ip netns exec private-router ip link set ss-router-gw up
[root@docker-net127 ~]# ip netns exec private-router ip link set ss-router-gw mtu 1500 state up
[root@docker-net127 ~]# ip link set ss-router-out mtu 1500 up
[root@docker-net127 ~]# brctl addif public-bridge ss-router-out
[root@docker-net127 ~]# ip netns exec private-router ip addr add dev ss-router-gw 200.160.0.3/24 \
broadcast 200.160.0.255
[root@docker-net127 ~]# ip netns exec private-router ip route add default via 200.160.0.1 dev \
ss-router-gw
[root@docker-net127 ~]# ip netns exec private-router ping -c 1 200.160.0.1
```
(5)根据租户分配的vxlan网络,在网络节点上创建对应vtep(vxlan网络端点)设备,以内网网络传输隧道数据
```
VXLan网络VNI - 此处以100为例
网络节点 - docker-net127
宿主内网网络设备(ens160,vlan610) - ens160.610
关闭防火墙/SELINUX允许后续vtep通信
```
```
[root@docker-net127 ~]# ip link add vxlan-100 type vxlan id 100 dstport 0 dev ens160.610
[root@docker-net127 ~]# brctl addif private-bridge vxlan-100
```
备注:租户网络的网络设备mtu全部1450

#### 第三节 各个计算节点上租户docker容器与kvm虚拟机的创建
(1) 创建docker容器
```
名字             宿主             网络信息
vxlan100-C1     comp126         IP:192.168.100.3/24
                                GW:192.168.100.1
                                CIDR:192.168.100.0/24
                                MAC-76:90:8e:b5:8a:27
vxlan100-C2     comp154         IP:192.168.100.4/24
                                GW:192.168.100.1
                                CIDR:192.168.100.0/24
                                MAC-1a:15:ec:78:9c:b3
vxlan100-C3     comp126         IP:192.168.100.5/24
                                GW:192.168.100.1
                                CIDR:192.168.100.0/24
                                MAC-0e:49:e8:8a:9e:e7
public-C2         comp154       IP:200.160.0.6/24
                                GW:200.160.0.1
                                CIDR:200.160.0.0/24
                                MAC-76:90:8e:b5:8a:aa
```
以public-C2为例,指定网络信息启动容器,并将分配的网络信息持久化到labels中; 除网络信息外也可以指定其他的选项
```
[root@docker ~]# docker run -idt --name public-C2 \
                 --label net_interface=eth0 \
                 --label net_ipaddress=200.160.0.6 \
                 --label net_mac=76:90:8e:b5:8a:aa \
                 --label net_prefix=24 \
                 --label net_gateway=200.160.0.1 \
                 --label net_cidr=200.160.0.0/24 \
                 --label net_bridge=br-public154 \
                 --net=none centos7 /bin/bash
```
使用脚本dknet为容器初始化网络; docker-net脚本参考附录链接
```
[root@docker ~]# sh -x docker-net public-C2
```
```
备忘:
1. 后续需要在配置文件中指定vxlan/vlan网络使用的物理网卡;e.g: net_host_phys_interface=ens160.610
2. L2代理进程维护vxlan-XX/vlan-XX设备及其与bridge的连接;
   e.g: net_type=vxlan,net_vxlan_id=10; 又 e.g: net_type=vlan,net_vlan_id=200
3. docker容器挂载的网桥由dknet自动创建,注意租户网络设备的mtu全部1450(包括docker容器网卡)
```
(2) 创建kvm虚拟机
```
名字            宿主         网络信息
vxlan100-C4    comp126      IP:192.168.100.6/24
                            GW:192.168.100.1
                            MAC-52:54:00:7c:d6:2c
public-C1      comp126      IP:200.160.0.7/24
                            GW:200.160.0.1
                            MAC-52:54:00:c6:3f:56
```
以vxlan100-C4为例,在宿主下为虚拟机创建数据目录,拷贝模板,连接到指定linux bridge
```
[root@docker-comp126 ~]# mkdir -p /data/instance/vxlan100-C4
[root@docker-comp126 ~]# cp /data/image/centos7.qcow2 /data/instance/vxlan100-C4/system
[root@docker-comp126 ~]# virt-install --name vxlan100-C4 \
--description vxlan100-C4 \
--ram  4096 \
--vcpus 4 --cpu host-model --accelerate --hvm \
--network bridge:qbr126-100,model=virtio \
--disk /data/instance/vxlan100-C4/system,bus=virtio,cache=writeback,driver_type=qcow2,size=10 \
--boot hd,cdrom \
--graphics vnc,listen=0.0.0.0 --noautoconsole \
--input tablet,bus=usb
```
使用vncviewer客户端进入启动的虚拟机,配置ip信息
```
[root@localhost ~]# ip addr add dev eth0 192.168.100.6/24
[root@localhost ~]# ip link set eth0 mtu 1450
[root@localhost ~]# ip route add default via 192.168.100.1 dev eth0
```
kvm挂载的网桥需要手动创建(后期脚本实现),注意租户网络设备的mtu全部1450(包括kvm虚拟机网卡);
计算节点租户网桥需要连接当前租户在当前宿主的vxlan vtep;
vtep的创建参考网络节点vxlan-100的配置,并将vtep设备连接到租户网桥上;

#### 第四节 各节点bridge添加fdb entry实现跨宿主租户内网连通
```
为保证网络畅通，需要从以下三方面进行配置:
1. 为网络节点的租户网络网桥添加fdb entry,处理arp包(目标mac为00:00:00:00:00:00,通过vtep设备forward到各个计算节点租户vtep)
2. 为网络节点的租户网络网桥添加fdb entry,处理单播包(目标明确,通过vtep设备forward到指定计算节点租户vtep)
3. 为各个计算节点的租户网络网桥添加fdb entry,处理arp包(目标mac为00:00:00:00:00:00,通过vtep设备forward到网络节点以及除自己外的租户业务宿主vtep)
```
备注:fdb添加entry的语法格式,如下
```
bridge fdb {add|append|del|replace} LLADDR dev DEV {local|temp} {self} {router} \
[dst IPADDR] [vni VNI] [port PORT] [via DEVICE]
```
具体参数说明请 man bridge fdb

此处举例说明(此处仅举例说明,并非详细配置):
1. 网络节点(10.160.0.127) - 从网络节点租户私有网络网关(初次)访问租户业务(192.168.100.6),MAC未知的情况:广播到该租户业务分布的业务宿主(10.160.0.126,10.160.0.154)
```
bridge fdb add 00:00:00:00:00:00 dev vxlan-100 dst 10.160.0.126
bridge fdb add 00:00:00:00:00:00 dev vxlan-100 dst 10.160.0.154
```
2. 网络节点(10.160.0.127) - 从网络节点租户私有网络网关访问租户业务(192.168.100.6),MAC已知的情况:单播到该租户业务所在宿主(10.160.0.126)
```
bridge fdb replace 52:54:00:7c:d6:2c dev vxlan-66 dst 10.160.0.126
```
3. 计算节点(10.160.0.126) - 从业务(192.168.100.6)访问同租户私有网络其他设备(192.168.100.1),
   MAC未知的情况:广播到该租户业务分布的业务宿主(当前宿主除外,10.160.0.154)以及网络节点(10.160.0.127)
```
bridge fdb add 00:00:00:00:00:00 dev vxlan-100 dst 10.160.0.127
bridge fdb add 00:00:00:00:00:00 dev vxlan-100 dst 10.160.0.154
```
4. 计算节点(10.160.0.154) - 从某业务(e.g:web)访问同租户其他业务或设备(e.g:db),MAC已知的情况:单播到目标业务所在宿主
```
bridge fdb replace 0e:49:e8:8a:9e:e7 dev vxlan-100 dst 10.160.0.126
```
```
考虑到:
(1)添加fdb entry工作的浩繁工作量
(2)人工操作的不可靠性
(3)依靠中控程序下刷并实时更新fdb entry可能引发单点故障 
由此设想:
各个计算节点拥有自己的agent程序,职责如下:
(1)作为可能的mesos slave的executor程序,负责具体业务及其附属网络设施的创建
(2)维护基础租户网络 vxlan-XX/bridge设备
(3)在新计算节点加入集群并创建某租户业务时,通过消息队列向同租户业务所在各个宿主宣告自己的诞生
(4)负责监听各租户新加入业务的消息,更新fdb entry,保证新业务所在私有网络各个节点互通
(5)在某租户在某宿主上的最后一个业务删除时,通过消息队列向同租户其他业务宣告自己的终结,并清除租户vxlan-XX/bridge设备
(6)负责监听各个租户删除业务的消息,更新fdb entry,保证之后租户内arp不会向已删除业务所在的宿主进行广播
(7)对外提供API:业务生命周期(e.g:业务重启后的网络重建)管理,镜像管理,存储管理
这样就可以实现多个agent之间的实时/异步互动,而中控程序主要负责业务创建的调用以及业务信息保存查询;且可以实现集群自管理,更易于人员维护;且去中心化,横向扩展能力更好
```


#### 第五节 为租户内网IP绑定浮动IP
(1)添加租户内网ip与浮动ip的nat映射
下面以192.168.100.6绑定浮动ip200.160.0.4为例
```
[root@docker-net127 ~]# ip netns exec private-router ip addr add 200.160.0.4/32 \
broadcast 200.160.0.4  dev ss-router-gw
[root@docker-net127 ~]# ip netns exec private-router iptables -t nat \
-A PREROUTING -d  200.160.0.4 -j DNAT --to-destination 192.168.100.6
[root@docker-net127 ~]# ip netns exec private-router iptables -t nat \
-A OUTPUT -d 200.160.0.4 -j DNAT --to-destination 192.168.100.6
[root@docker-net127 ~]# ip netns exec private-router iptables -t nat \
-A POSTROUTING -s 192.168.100.6 -j SNAT --to-source 200.160.0.4
```
(2)公网测试web服务以及ssh可用性
```
[root@net ~]# curl 200.160.0.4:9898
This Page is got from server 192.168.100.6(nat to 200.160.0.4)
[root@net ~]# ssh root@200.160.0.4
root@200.160.0.4's password:
Last login: Sun Nov 20 16:47:51 2016 from 200.160.0.1
[root@localhost ~]#
```
#### 第六节 负载均衡即服务实现
(1)添加负载均衡前端ip与设备
```
[root@docker-net127 ~]# ip link add name ha-if type veth peer name ha-out
[root@docker-net127 ~]# ip link set ha-if mtu 1450
[root@docker-net127 ~]# brctl addif private-bridge ha-out
[root@docker-net127 ~]# ip netns add ha-vxlan100-web
[root@docker-net127 ~]# ip link set ha-if netns ha-vxlan100-web
[root@docker-net127 ~]# ip netns exec ha-vxlan100-web ip link set lo up
[root@docker-net127 ~]# ip link set ha-out mtu 1450 up
[root@docker-net127 ~]# ip netns exec ha-vxlan100-web ip addr add dev ha-if 192.168.100.8/24
[root@docker-net127 ~]# ip netns exec ha-vxlan100-web ip route add default via 192.168.100.1 dev ha-if
```
(2)在vxlan网络(某租户网络)内启动负载均衡服务
首先, 编辑haproxy配置文件:
```
[root@docker-net127 ~]# mkdir -p /var/lib/netns/ha-vxlan100-web/
[root@docker-net127 ~]# cat > /var/lib/netns/ha-vxlan100-web/haproxy.cfg << EOF
global
   daemon
   user nobody
   group haproxy
   log /dev/log local0
   log /dev/log local1 notice
defaults
   log global
   retries 3
   option redispatch
   timeout connect 5000
   timeout client 50000
   timeout server 50000
frontend c4c49fc1-bf10-4270-bc20-02b93439897f
   option tcplog
   bind 192.168.100.8:80
   mode http
   default_backend 8f09dbaf-0c78-48f1-8b80-662e75036485
   maxconn 100
   option forwardfor
backend 8f09dbaf-0c78-48f1-8b80-662e75036485
   mode http
   balance leastconn
   option forwardfor
   server 0c7d3741-df7b-4bf3-9396-14cc3c05b5a5 192.168.100.6:9898 weight 100
   server 44c18162-301e-42f3-9f0a-5522f426412f 192.168.100.7:9898 weight 100
EOF
```
然后, 按指定配置文件在指定网络命名空间下启动haproxy服务:
```
[root@docker-net127 ~]# ip netns exec ha-vxlan100-web haproxy -f /var/lib/netns/ha-vxlan100-web/haproxy.cfg
```

(3)租户内网测试负载均衡服务
```
[root@docker-net127 ~]# ip netns exec private-router curl 192.168.100.8
This Page is got from server 192.168.100.6(nat to 200.160.0.4)
[root@docker-net127 ~]# ip netns exec private-router curl 192.168.100.8
This page is got from 192.168.100.7
```

(4)为负载均衡前端ip绑定浮动ip(200.160.0.5),对公网提供服务
```
[root@docker-net127 ~]# ip netns exec private-router ip addr add 200.160.0.5/32
broadcast 200.160.0.5  dev ss-router-gw
[root@docker-net127 ~]# ip netns exec private-router iptables -t nat \
-A PREROUTING -d 200.160.0.5 -j DNAT --to-destination 192.168.100.8
[root@docker-net127 ~]# ip netns exec private-router iptables -t nat \
-A OUTPUT -d 200.160.0.5 -j DNAT --to-destination 192.168.100.8
[root@docker-net127 ~]# ip netns exec private-router iptables -t nat \
-A POSTROUTING -s 192.168.100.8 -j SNAT --to-source 200.160.0.5
```
(5)公网测试负载均衡服务
```
[root@net ~]# curl 200.160.0.5
This Page is got from server 192.168.100.6(nat to 200.160.0.4)
[root@net ~]# curl 200.160.0.5
This page is got from 192.168.100.7
```
#### 附录
```
1.docker-net脚本代码
```
```
#!/bin/bash    
# Author guojingyu    
# Email guojy8993@163.com    
# Date 2016/10/24    
# Desc Start docker container and setup network    
iface_max_len=15    
brg_max_len=13
    
# @param ${1} container    
function check_docker(){    
exists=$(docker ps -a | egrep "^${1}\s+|\s+${1}$" | wc -l)    
}    
# @param ${1} container    
function start_docker(){    
check_docker ${1}    
[ ${exists} -eq 0 ] && echo "ERROR: Container ${1} does not exist" && exit 1    
docker start ${1} 1>/dev/null 2>/dev/null    
[ $? -gt 0 ] && echo "ERROR: Fail to start Container ${1}" && exit 1    
}
    
# @param ${1} container    
function parse_labels(){    
interface=$(docker inspect -f '{{.Config.Labels.net_interface}}' ${1})    
ipaddr=$(docker inspect -f '{{.Config.Labels.net_ipaddress}}' ${1})    
prefix=$(docker inspect -f '{{.Config.Labels.net_prefix}}' ${1})    
gateway=$(docker inspect -f '{{.Config.Labels.net_gateway}}' ${1})    
bridge=$(docker inspect -f '{{.Config.Labels.net_bridge}}' ${1})    
mac=$(docker inspect -f '{{.Config.Labels.net_mac}}' ${1})    
cidr=$(docker inspect -f '{{.Config.Labels.net_cidr}}' ${1})    
} 
   
# @param ${1} bridge    
function ensure_bridge(){    
brctl --help 1>/dev/null 2>/dev/null    
[ $? -gt 0 ] && echo "ERROR: Rpm bridge-util required !" && exit 1    
count=$(brctl show | grep ${1} | wc -l)    
[ ${count} -eq 0 ] && brctl addbr ${1} && ip link set ${1} mtu 1450 up    
} 
   
# @param ${1} container/netns    
function setup_netns(){    
[ $(ip netns | grep "^${1}$" | wc -l) -eq 1 ] && ip netns del ${1}    
pid=$(docker inspect -f '{{.State.Pid}}' ${1})    
mkdir -p /var/run/netns && ln -s /proc/${pid}/ns/net /var/run/netns/${1}    
}
    
# @param ${1} container/netns    
# @param ${2} netdev    
# @param ${3} ip    
# @param ${4} prefix    
# @param ${5} gateway    
function setup_ipaddr(){    
# ip netns exec ${1} ip addr flush dev ${2}    
ip netns exec ${1} ip addr add dev ${2} ${3}/${4}    
# ip netns exec ${1} ip route flush dev ${2}    
ip netns exec ${1} ip route add default via ${5} dev ${2}    
}
    
# @param ${1} bridge    
# @param ${2} docker/netns    
# @param ${3} netdev    
# @param ${4} mac    
function ensure_link(){    
veth_master_device="co-${2}"    
veth_slave_device="ci-${2}"    
master=${veth_master_device:0:${iface_max_len}}    
slave=${veth_slave_device:0:${iface_max_len}}    
ip link del dev ${master} 2>/dev/null    
ip link add ${master} type veth peer name ${slave}    
ip link set ${slave} mtu 1450 netns ${2}    
# set docker lo/ethX(mtu/state)    
ip netns exec ${2} ip link set lo up    
ip netns exec ${2} ip link set ${slave} name ${3}    
ip netns exec ${2} ip link set dev ${3} address ${4} up    
ip link set ${master} mtu 1450 up    
brctl show ${1} | grep ${master} | wc -l    
[ $? -eq 0 ] && brctl addif ${1} ${master}    
} 
   
function build_network(){    
setup_netns ${1}    
ensure_bridge ${2}    
ensure_link ${2} ${1} ${interface} ${mac}    
setup_ipaddr ${1} ${interface} ${ipaddr} ${prefix} ${gateway}    
} 
   
# @param ${1} container/netns    
function setup()    
{    
start_docker ${1}    
parse_labels ${1}    
build_network ${1} ${bridge}    
}    
setup ${1}
```
```
本文系原创文章,转载请注明作者 ; 如有疑问请联系 guojy8993@163.com
```
