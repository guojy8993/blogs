### 文档内容提要 ###

1.网络节点虚拟交换机设置

2.网络节点租户网络路由与dncp的设置

3.计算节点虚拟交换机设置以及虚拟机的启动

4.网络与计算节点交换机流表的设置

5.测试修改dhcp配置允许指定mac的虚拟机dhcp获取指定ip

6.租户虚拟机绑定外网IP

7.负载均衡实现

### 预期目标 ###

1.虚拟机与虚拟路由网络联通

2.跨宿主同租户虚拟机之间网络联通

3.虚拟机与外网通信

4.虚拟机web业务负载均衡

5.租户隔离



### 测试环境需求说明 ###
```
节点名称   节点角色     网络                            系统软件环境
---------------------------------------------------------------------------------
net117     网络节点    ens160 外网 10.160.0.117/16     openvswitch & bridge-utils
                      ens224 外网 -(配置vlan)-
                      ens192 隧道 172.172.172.117/24
---------------------------------------------------------------------------------
comp115    计算节点    ens160 外网 10.160.0.115/16     openvswitch & bridge-utils
                      ens192 隧道 172.172.172.115/24         & 虚拟化环境
---------------------------------------------------------------------------------
comp114    计算节点    ens160 外网 10.160.0.114/16     openvswitch & bridge-utils
                      ens192 隧道 172.172.172.114/24         & 虚拟化环境
---------------------------------------------------------------------------------
```

### Demo租户在某server上的本地vlan与租户全局vni的分配 ###
```
---------------------------------------------------------------------------------
Server    LOCAL_VLAN_ID   VXLAN_ID
---------------------------------------------------------------------------------
net117    300             8888
---------------------------------------------------------------------------------
comp115   100             8888
---------------------------------------------------------------------------------
comp114   200             8888
---------------------------------------------------------------------------------
```

_ _ _

#### 网络节点设置 ####

(1) 虚拟交换机的设置

网络节点网卡/IP/标签 说明:

ens160  配置610vlan,ip 10.160.0.117/16通办公网作为外网(internal标签)使用,后同此例.

ens192  配置172.172.172.117/24作为隧道网络

ens224  同ens160,作为虚拟机走网络节点通外网的桥接设备

创建3个必须的ovs交换机:

```
[root@net117 ~]# ovs-vsctl add-br br-ex
[root@net117 ~]# ovs-vsctl add-br br-int
[root@net117 ~]# ovs-vsctl add-br br-tun
```

在外网网卡上创建vlan610,并桥接到br-ex上
```
[root@net117 ~]# ip link add link ens224 name ens224.610 type vlan id 610
[root@net117 ~]# ip link set ens224.610 up
[root@net117 ~]# ovs-vsctl add-port br-ex ens224.610
```

br-int 通过patch port与br-tun 互联

```
[root@net117 ~]# ovs-vsctl add-port br-int patch-tun -- set Interface patch-tun type=patch options:peer=patch-int
[root@net117 ~]# ovs-vsctl add-port br-tun patch-int -- set Interface patch-int type=patch options:peer=patch-tun \
-- set Interface patch-int ofport_request=1
```

将br-tun通过隧道ip与各个计算节点建立传输隧道

```
net117   172.172.172.117
comp115  172.172.172.115
comp114  172.172.172.114
```
```
[root@net117 ~]# ovs-vsctl -- --if-exists del-port tun2comp115 -- add-port br-tun tun2comp115 \
-- set Interface tun2comp115 type=vxlan options:local_ip=172.172.172.117 options:remote_ip=172.172.172.115 options:key=flow

[root@net117 ~]# ovs-vsctl -- --if-exists del-port tun2comp114 -- add-port br-tun tun2comp114 \
-- set Interface tun2comp114 type=vxlan options:local_ip=172.172.172.117 options:remote_ip=172.172.172.114 options:key=flow
```

vxlan使用udp:4789通信

```
[root@net117 ~]# iptables -I INPUT -p udp --dport 4789 -j ACCEPT
```


(2) DHCP命名空间的设置

创建dhcp设备:dnsmasq进程监听的interface

```
[root@net117 ~]# ovs-vsctl -- --if-exists del-port private-dhcp -- add-port br-int private-dhcp \
-- set Interface private-dhcp type=internal \
-- set Interface private-dhcp external-ids:iface-id=733431fe-567a-447f-a29e-5fb94aa36a4c \
-- set Interface private-dhcp external-ids:iface-status=active \
-- set Interface private-dhcp external-ids:attached-mac=fa:16:00:4c:74:16

[root@net117 ~]# ovs-vsctl set Port private-dhcp tag=300
```

创建dhcp命名空间,并将上述设备移入其中

```
[root@net117 ~]# ip netns add ns-private-dhcp
[root@net117 ~]# ip link set private-dhcp netns ns-private-dhcp
[root@net117 ~]# ip netns exec ns-private-dhcp ip link set lo up
[root@net117 ~]# ip netns exec ns-private-dhcp ip link set lo state up
[root@net117 ~]# ip netns exec ns-private-dhcp ip link set private-dhcp up
[root@net117 ~]# ip netns exec ns-private-dhcp ip link set private-dhcp state up
```

租户子网资源池 150.150.150.0/24,dnsmasq监听设备private-dhcp(150.150.150.2)
```
[root@net117 ~]# ip netns exec ns-private-dhcp ip addr add 150.150.150.2/24 dev private-dhcp
[root@net117 ~]# ip netns exec ns-private-dhcp ip route add default via 150.150.150.1 dev private-dhcp
```

启动dnsmasq进程,对租户提供ip资源池

```
[root@net117 ~]# uuidgen
d30a8588-f968-4f62-9388-4ed8f61e8355
[root@net117 ~]# mkdir -p /var/lib/dhcp/d30a8588-f968-4f62-9388-4ed8f61e8355
```

设置子网设备mtu 1450,资源池网关

```
[root@net117 ~]# echo "dhcp-option-force=26,1454" > /var/lib/dhcp/d30a8588-f968-4f62-9388-4ed8f61e8355/mtu
[root@net117 ~]# echo "tag:tag0,option:router,150.150.150.1" > /var/lib/dhcp/d30a8588-f968-4f62-9388-4ed8f61e8355/opts
```

在dhcp命名空间下启动dnsmasq服务

```
[root@net117 ~]# ip netns exec ns-private-dhcp dnsmasq --no-hosts \
  --no-resolv --strict-order --bind-interfaces --interface=private-dhcp \
  --except-interface=lo \
  --pid-file=/var/lib/dhcp/d30a8588-f968-4f62-9388-4ed8f61e8355/pid \
  --dhcp-hostsfile=/var/lib/dhcp/d30a8588-f968-4f62-9388-4ed8f61e8355/host \
  --addn-hosts=/var/lib/dhcp/d30a8588-f968-4f62-9388-4ed8f61e8355/addn_hosts \
  --dhcp-optsfile=/var/lib/dhcp/d30a8588-f968-4f62-9388-4ed8f61e8355/opts \
  --dhcp-leasefile=/var/lib/dhcp/d30a8588-f968-4f62-9388-4ed8f61e8355/leases \
  --dhcp-range=set:tag0,150.150.150.0,static,86400s \
  --dhcp-lease-max=256 \
  --conf-file=/var/lib/dhcp/d30a8588-f968-4f62-9388-4ed8f61e8355/mtu \
  --domain=devops
```

3.虚拟路由的创建

创建租户子网网关设备

```
[root@net117 ~]# ovs-vsctl -- --if-exists del-port private-router -- add-port br-int private-router \
-- set Interface private-router type=internal \
-- set Interface private-router external-ids:iface-id=633432fe-567a-447f-a29e-5fb94aa36a4c \
-- set Interface private-router external-ids:iface-status=active \
-- set Interface private-router external-ids:attached-mac=fa:16:00:4d:74:16

[root@net117 ~]# ovs-vsctl set Port private-router tag=300
```

创建虚拟路由命名空间
```
[root@net117 ~]# ip netns add ns-private-router
[root@net117 ~]# ip link set private-router netns ns-private-router
[root@net117 ~]# ip netns exec ns-private-router ip link set lo up
[root@net117 ~]# ip netns exec ns-private-router ip link set lo state up
[root@net117 ~]# ip netns exec ns-private-router ip link set private-router up
[root@net117 ~]# ip netns exec ns-private-router ip link set private-router state up
```

配置内网网关

```
[root@net117 ~]# ip netns exec ns-private-router ip addr add dev private-router 150.150.150.1/24
```

在br-ex上创建路由外网网关设备,并置于虚拟路由之中

```
[root@net117 ~]# ovs-vsctl -- --if-exists del-port extgw -- add-port br-ex extgw \
-- set Interface extgw type=internal \
-- set Interface extgw external-ids:iface-id=893431fe-567a-447f-d59e-5fb94aa36a40 \
-- set Interface extgw external-ids:iface-status=active \
-- set Interface extgw external-ids:attached-mac=fa:16:00:4c:88:16
[root@net117 ~]# ip link set extgw netns ns-private-router
[root@net117 ~]# ip netns exec ns-private-router ip link set extgw up
[root@net117 ~]# ip netns exec ns-private-router ip link set extgw state up
[root@net117 ~]# ip netns exec ns-private-router ip addr add dev extgw 10.160.0.148/16 broadcast 10.160.255.255
[root@net117 ~]# ip netns exec ns-private-router ip route add default via 10.160.0.1 dev extgw
```

在虚拟路由内测试访问外网是否网络畅通

```
[root@net117 ~]# ip netns exec ns-private-router ping -c 1 10.160.0.1
PING 10.160.0.1 (10.160.0.1) 56(84) bytes of data.
64 bytes from 10.160.0.1: icmp_seq=1 ttl=255 time=2.20 ms
...
```

_ _ _

##### 计算节点配置(以comp115为例) #####

添加br-tun与br-int交换机
```
[root@comp115 ~]# ovs-vsctl add-br br-int
```

清除br-tun默认转发规则,避免net117/comp114/comp115之间vxlan隧道环路
```
[root@comp115 ~]# ovs-vsctl add-br br-tun && ovs-vsctl del-flows br-tun
```

创建patch ports将br-tun与br-int互联

```
[root@comp115 ~]# ovs-vsctl add-port br-int patch-tun \
-- set Interface patch-tun type=patch options:peer=patch-int

[root@comp115 ~]# ovs-vsctl add-port br-tun patch-int \
-- set Interface patch-int type=patch options:peer=patch-tun
```

为br-tun添加隧道传输端口

net117   172.172.172.117

comp115  172.172.172.115

comp114  172.172.172.114

```
[root@comp115 ~]# ovs-vsctl -- --if-exists del-port tun2net117 -- add-port br-tun tun2net117 \
-- set Interface tun2net117 type=vxlan options:local_ip=172.172.172.115  options:remote_ip=172.172.172.117 options:key=flow

[root@comp115 ~]# ovs-vsctl -- --if-exists del-port tun2comp114 -- add-port br-tun tun2comp114 \
-- set Interface tun2comp114 type=vxlan options:local_ip=172.172.172.115 options:remote_ip=172.172.172.114 options:key=flow
```

说明:

vxlan使用udp:4789通信
```
[root@comp115 ~]# iptables -I INPUT -p udp --dport 4789 -j ACCEPT
```

创建linux bridge,未来作为安全组防火墙实现之处

创建对等设备,将其与br-int互联

```
[root@comp115 ~]# ip link add veth-sg-br type veth peer name veth-br-int
[root@comp115 ~]# ip link set veth-sg-br up
[root@comp115 ~]# ip link set veth-br-int up
[root@comp115 ~]# brctl addbr brg-sg
[root@comp115 ~]# ip link set brg-sg up
[root@comp115 ~]# brctl addif brg-sg veth-sg-br
```

设置demo租户在comp115上的本地vlan为100 

设置demo租户在comp114上的本地vlan为200

设置demo租户在net117 上的本地vlan为300

设置demo租户vxlan的vni为8888

说明: 本地vlan的使用保证同宿主同租户业务互通,不同租户通过vlan隔离

```
[root@comp115 ~]# ovs-vsctl add-port br-int veth-br-int -- set Port veth-br-int tag=100
```

启动虚拟机连接到 firewall linux bridge 上

初始化虚拟化环境:
```
yum install -y fence-virtd-libvirt.x86_64 libvirt-daemon.x86_64 libvirt-devel.x86_64 \
               libvirt.x86_64 libvirt-daemon-kvm.x86_64 libvirt-daemon-lxc.x86_64 \
	       libvirt-daemon-driver-qemu.x86_64 qemu-guest-agent.x86_64 qemu-img.x86_64 \
	       qemu-kvm.x86_64  qemu-kvm-common.x86_64 qemu-kvm-tools.x86_64 \
	       fence-agents-virsh.x86_64 virt-install
systemctl start  libvirtd
systemctl enable libvirtd
```
```
[root@comp115 ~]# cp /data/images/centos7.qcow2 /data/instance/comp115-demo0x01/system
[root@comp115 ~]# cat > install << EOF
virt-install --name comp114-demo0x01 --ram 4096 --vcpus 4 \
             --boot hd \
             --disk /data/instance/comp114-demo0x01/system,format=qcow2 \
             --network bridge:brg-sg,model=virtio \
             --graphics vnc,listen=0.0.0.0 --noautoconsole
EOF
```

查看新建虚拟机的vnc端口号

说明: iptables放开vnc端口访问
```
[root@comp115 ~]# iptables -I INPUT -p tcp --dport 5900:6900 -j ACCEPT
[root@comp115 ~]# virsh vncdisplay comp115-demo0x01
:0
```

使用vncviewer连接宿主(comp115)端口(5900+0)查看视图,进行管理

配置ip(也可以使用dhcp方式,从网络节点的qdhcp处获取)

ping测试租户路由内网网关是否通

```
[root@localhost ~]# ip addr add dev eth0 150.150.150.4/24
[root@localhost ~]# ip route add default via 150.150.150.1 dev eth0
[root@localhost ~]# ip link set eth0 mtu 1450
```
同上所示,在comp114上创建comp114-demo0x01,配置150.150.150.3/24

_ _ _

#### 计算节点流表配置与说明 ####

(1)table0: 对流量按方向进行分发:

a. 虚拟机出流量(从patch-int进入br-tun)提交table 1处理

b. 从net117/compXXX隧道口进来的流量提交table 3处理

以comp115为例:

table 0 默认DROP
```
table=0,priority=0 action=DROP
```
使用"ovs-ofctl show br-tun"查看net117以及comp114,patch-int对应的ovs端口号:
tun2net117 -> 5
tun2comp114 -> 4
patch-int -> 1

```
table=0,priority=1,in_port=1 action=resubmit(,1)
table=0,priority=1,in_port=5 action=resubmit(,3)
table=0,priority=1,in_port=4 action=resubmit(,3)
```

(2)table1: 对出虚拟机的流量按是否是广播进行分类处理

a. 广播包发送到table21处理
b. 单播包发送到table20处理
说明: 要么是单播,要么是广播,没有例外情况,故而无需默认流表
```
table=1,priority=1,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 action=resubmit(,21)
table=1,priority=1,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 action=resubmit(,20)
```
(3)table3: 对于从隧道进入的流量进行处理

默认DROP
```
table=3,priority=0 action=DROP
```

针对有业务调度到当前宿主上的租户,全部添加规则,实现租户vxlan_id -> 本地vlan_id的转换

以demo用户全局vxlan_id 8888,在comp115上的本地vlan_id 100为例
```
table=3,priority=1,tun_id=8888 action=mod_vlan_vid:100,resubmit(,10)
```

如果还有其他租户的业务,如上所示继续添加流表

(4)table10: 督促table20添加流表,学习对"从特定端口进入,当前携带特定vlan,特定源mac的流量"的处理(见下),并输出到patch-int(端口号1).
处理动作包括:
a.去除本地vlan
b.并换上全局vxlan_id
c.从请求包的交换机入端口再输出回去
```
table=10,priority=1 action=learn(table=20,hard_timeout=300,priority=1,\
NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],\
load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:1
```
(5)实现虚拟机对外单播的处理
a.根据学习规则的规则处理
<...此处是学习到的规则s !!! ..>
b.如果数据包因为之前学习的规则自然老化而匹配不到,则提交到table21进行广播
```
table=20,priority=0 action=resubmit(,21)
```

(6)实现虚拟机对外广播的处理

a.默认DROP

```
table=21,priority=0 action=DROP
```

b.针对有业务调度到当前宿主上的租户,全部添加规则,实现本地vlan_id -> 租户全局vxlan_id 的转换,并发送到去往网络节点(net117)以及其他计算节点(comp114)的隧道口

```
table=21,priority=1,dl_vlan=100,actions=strip_vlan,set_tunnel:8888,output:4,output:5
```

如果还有其他租户的业务,如上所示继续添加流表

PS:流表测试

(1) 虚拟机arp出包,询问租户网络网关(特别注意: arp请求是作为单播处理的)

以虚拟机 comp115-demo0x01/150.150.150.4/52:54:00:22:09:1e 为例:

虚拟机出br-int端口,会添加本地vlan的vlan header(此时100)
```
[root@comp115 ~]# ovs-appctl ofproto/trace br-tun arp,arp_op=1,in_port=1,dl_src=52:54:00:22:09:1e,dl_vlan=100,arp_spa=150.150.150.4,arp_tpa=150.150.150.1
Bridge: br-tun
Flow: arp,in_port=1,dl_vlan=100,dl_vlan_pcp=0,dl_src=52:54:00:22:09:1e,dl_dst=00:00:00:00:00:00,arp_spa=150.150.150.4,\
arp_tpa=150.150.150.1,arp_op=0,arp_sha=00:00:00:00:00:00,arp_tha=00:00:00:00:00:00

Rule: table=0 cookie=0 priority=1,in_port=1
OpenFlow actions=resubmit(,1)

Resubmitted flow: arp,in_port=1,dl_vlan=100,dl_vlan_pcp=0,dl_src=52:54:00:22:09:1e,dl_dst=00:00:00:00:00:00,\
arp_spa=150.150.150.4,arp_tpa=150.150.150.1,arp_op=0,arp_sha=00:00:00:00:00:00,arp_tha=00:00:00:00:00:00
Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0
Resubmitted  odp: drop
Resubmitted megaflow: recirc_id=0,arp,in_port=1,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00
Rule: table=1 cookie=0 priority=1,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00
OpenFlow actions=resubmit(,20)

Resubmitted flow: unchanged
Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0
Resubmitted  odp: drop
Resubmitted megaflow: recirc_id=0,arp,in_port=1,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00
Rule: table=20 cookie=0 priority=0
OpenFlow actions=resubmit(,21)

Resubmitted flow: unchanged
Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0
Resubmitted  odp: drop
Resubmitted megaflow: recirc_id=0,arp,in_port=1,dl_vlan=100,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00
Rule: table=21 cookie=0 priority=1,dl_vlan=100
OpenFlow actions=strip_vlan,set_tunnel:0x22b8,output:4,output:5
output to kernel tunnel
output to kernel tunnel

Final flow: arp,tun_id=0x22b8,in_port=1,vlan_tci=0x0000,dl_src=52:54:00:22:09:1e,dl_dst=00:00:00:00:00:00,\
arp_spa=150.150.150.4,arp_tpa=150.150.150.1,arp_op=0,arp_sha=00:00:00:00:00:00,arp_tha=00:00:00:00:00:00
Megaflow: recirc_id=0,arp,in_port=1,dl_vlan=100,dl_vlan_pcp=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00
Datapath actions: set(tunnel(tun_id=0x22b8,src=172.172.172.115,dst=172.172.172.114,ttl=64,flags(df|key))),\
pop_vlan,3,set(tunnel(tun_id=0x22b8,src=172.172.172.115,dst=172.172.172.117,ttl=64,flags(df|key))),3
```

(2) 网关arp应答

网关:   private-router/150.150.150.1/72:04:4d:fc:be:76
虚拟机: comp115-demo0x01/150.150.150.4/52:54:00:22:09:1e

```
[root@comp115 ~]# ovs-appctl ofproto/trace br-tun arp,arp_op=2,in_port=5,tun_id=8888,arp_sha=72:04:4d:fc:be:76,\
arp_tha=52:54:00:22:09:1e,arp_spa=150.150.150.1,arp_tpa=150.150.150.4,dl_src=72:04:4d:fc:be:76,dl_dst=52:54:00:22:09:1e -generate
Bridge: br-tun
Flow: arp,tun_id=0x22b8,in_port=5,vlan_tci=0x0000,dl_src=72:04:4d:fc:be:76,dl_dst=52:54:00:22:09:1e,arp_spa=150.150.150.1,\
arp_tpa=150.150.150.4,arp_op=2,arp_sha=72:04:4d:fc:be:76,arp_tha=52:54:00:22:09:1e

Rule: table=0 cookie=0 priority=1,in_port=5
OpenFlow actions=resubmit(,3)

Resubmitted flow: arp,tun_id=0x22b8,in_port=5,vlan_tci=0x0000,dl_src=72:04:4d:fc:be:76,dl_dst=52:54:00:22:09:1e,\
arp_spa=150.150.150.1,arp_tpa=150.150.150.4,arp_op=2,arp_sha=72:04:4d:fc:be:76,arp_tha=52:54:00:22:09:1e
Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0
Resubmitted  odp: drop
Resubmitted megaflow: recirc_id=0,arp,tun_id=0x22b8,in_port=5
Rule: table=3 cookie=0 priority=1,tun_id=0x22b8
OpenFlow actions=mod_vlan_vid:100,resubmit(,10)

Resubmitted flow: arp,tun_id=0x22b8,in_port=5,dl_vlan=100,dl_vlan_pcp=0,dl_src=72:04:4d:fc:be:76,dl_dst=52:54:00:22:09:1e,\
arp_spa=150.150.150.1,arp_tpa=150.150.150.4,arp_op=2,arp_sha=72:04:4d:fc:be:76,arp_tha=52:54:00:22:09:1e
Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0
Resubmitted  odp: drop
Resubmitted megaflow: recirc_id=0,arp,tun_id=0x22b8,in_port=5,vlan_tci=0x0000/0x1fff
Rule: table=10 cookie=0 priority=1
OpenFlow actions=learn(table=20,hard_timeout=300,priority=1,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],\
load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:1
...
```

验证table20学习表是否学习到处理规则

```
[root@comp115 ~]# ovs-ofctl dump-flows br-tun | grep "table=20" | grep 72:04:4d:fc:be:76
 cookie=0x0, duration=203.573s, table=20, n_packets=0, n_bytes=0, hard_timeout=300, idle_age=203, priority=1,\
 vlan_tci=0x0064/0x0fff,dl_dst=72:04:4d:fc:be:76 actions=load:0->NXM_OF_VLAN_TCI[],load:0x22b8->NXM_NX_TUN_ID[],output:5
```
```
[root@comp115 ~]# ovs-appctl ofproto/trace br-tun arp,arp_op=2,in_port=5,tun_id=8888,arp_sha=72:04:4d:fc:be:76,\
arp_tha=52:54:00:22:09:1e,arp_spa=150.150.150.1,arp_tpa=150.150.150.4,dl_src=72:04:4d:fc:be:76,dl_dst=52:54:00:22:09:1e -generate
```
(3)再测试虚拟机单播出包

此时的虚拟机单播出包根据学习到的规则直接处理

```
[root@comp115 ~]# ovs-appctl ofproto/trace br-tun ip,in_port=1,dl_src=52:54:00:22:09:1e,dl_dst=72:04:4d:fc:be:76,dl_vlan=100,nw_src=150.150.150.4,nw_dst=150.150.150.1
Bridge: br-tun
Flow: ip,in_port=1,dl_vlan=100,dl_vlan_pcp=0,dl_src=52:54:00:22:09:1e,dl_dst=72:04:4d:fc:be:76,nw_src=150.150.150.4,\
nw_dst=150.150.150.1,nw_proto=0,nw_tos=0,nw_ecn=0,nw_ttl=0

Rule: table=0 cookie=0 priority=1,in_port=1
OpenFlow actions=resubmit(,1)

Resubmitted flow: ip,in_port=1,dl_vlan=100,dl_vlan_pcp=0,dl_src=52:54:00:22:09:1e,dl_dst=72:04:4d:fc:be:76,nw_src=150.150.150.4,\
nw_dst=150.150.150.1,nw_proto=0,nw_tos=0,nw_ecn=0,nw_ttl=0
Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0
Resubmitted  odp: drop
Resubmitted megaflow: recirc_id=0,ip,in_port=1,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00,nw_frag=no
Rule: table=1 cookie=0 priority=1,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00
OpenFlow actions=resubmit(,20)

Resubmitted flow: unchanged
Resubmitted regs: reg0=0x0 reg1=0x0 reg2=0x0 reg3=0x0 reg4=0x0 reg5=0x0 reg6=0x0 reg7=0x0
Resubmitted  odp: drop
Resubmitted megaflow: recirc_id=0,ip,in_port=1,vlan_tci=0x0064/0x0fff,dl_dst=72:04:4d:fc:be:76,nw_frag=no
Rule: table=20 cookie=0 priority=1,vlan_tci=0x0064/0x0fff,dl_dst=72:04:4d:fc:be:76
OpenFlow actions=load:0->NXM_OF_VLAN_TCI[],load:0x22b8->NXM_NX_TUN_ID[],output:5
output to kernel tunnel
...
```

上述添加的流表显示虚拟机访问网络节点网关没有问题,待后续添加comp114之后同理测试同租户不同宿主业务之间的互访

comp114计算节点br-tun流表参考上文逻辑添加,流表规则如下:

端口对应关系:

patch-int -> 1

tun2net117 -> 2

tun2comp115 -> 3

租户demo的本地vlan 200,虚拟机comp114-demo0x01
生成流表:
```
[root@comp114 ~]# cat > tun_flows << EOF
table=0,priority=1,in_port=1 action=resubmit(,1)
table=0,priority=1,in_port=2 action=resubmit(,3)
table=0,priority=1,in_port=3 action=resubmit(,3)
table=0,priority=0 action=DROP
table=1,priority=1,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 action=resubmit(,21)
table=1,priority=1,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 action=resubmit(,20)
table=3,priority=1,tun_id=8888 action=mod_vlan_vid:200,resubmit(,10)
table=3,priority=0 action=DROP
table=10,priority=1 action=learn(table=20,hard_timeout=300,priority=1,NXM_OF_VLAN_TCI[0..11],\
NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],\
output:NXM_OF_IN_PORT[]),output:1
table=20,priority=0 action=resubmit(,21)
table=21,priority=1,dl_vlan=200,actions=strip_vlan,set_tunnel:8888,output:3,output:2
table=21,priority=0 action=DROP
EOF
```

批量添加流表:
```
[root@comp114 ~]# ovs-ofctl add-flows br-tun tun_flows
```
_ _ _

#### 网络节点流表配置与说明 ####
(1)table0: 对流量按方向进行分发

a. 网关到虚拟机流量(从patch-int进入br-tun)提交table 1处理

b. 从compXXX隧道口进来的流量提交table 3处理

net117上br-tun流表讲解:

table 0 默认DROP

```
table=0,priority=0 action=DROP
```

使用"ovs-ofctl show br-tun"查看comp114,comp115,patch-int对应的ovs端口号:

tun2comp115 -> 2

tun2comp114 -> 3

patch-int -> 1
```
table=0,priority=1,in_port=1 action=resubmit(,1)
table=0,priority=1,in_port=2 action=resubmit(,3)
table=0,priority=1,in_port=3 action=resubmit(,3)
```
(2)table1: 对出虚拟机的流量按是否是广播进行分类处理

a. 广播包发送到table21处理

b. 单播包发送到table20处理

说明: 要么是单播,要么是广播,没有例外情况,故而无需默认流表

```
table=1,priority=1,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 action=resubmit(,21)
table=1,priority=1,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 action=resubmit(,20)
```

(3)table3: 对于从隧道进入的流量进行处理

默认DROP
```
table=3,priority=0 action=DROP
```

针对有虚拟路由/dhcp存在于当前宿主上的租户,全部添加规则,实现租户vxlan_id -> 网络节点本地vlan_id的转换

以demo用户全局vxlan_id 8888,在net117上的本地vlan_id 300为例
```
table=3,priority=1,tun_id=8888 action=mod_vlan_vid:300,resubmit(,10)
```

如果还有其他租户的业务,如上所示继续添加流表

(4)table10: 督促table20添加流表,学习对"从特定端口进入,当前携带特定vlan,特定源mac的流量"的处理(见下),并输出到patch-int(端口号1)

处理动作包括:

a.去除本地vlan

b.并换上全局vxlan_id

c.从请求包的交换机入端口再输出回去

```
table=10,priority=1 action=learn(table=20,hard_timeout=300,priority=1,NXM_OF_VLAN_TCI[0..11],\
NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],\
output:NXM_OF_IN_PORT[]),output:1
```

(5)实现虚拟机对外单播的处理

a. 根据学习规则的规则处理

<...此处是学习到的规则s !!! ..>

b. 如果数据包因为之前学习的规则自然老化而匹配不到,则提交到table21进行广播

table=20,priority=0 action=resubmit(,21)

(6) 实现虚拟机对外广播的处理

a. 默认DROP

```
table=21,priority=0 action=DROP
```

b. 针对在当前宿主上有虚拟路由网关/dhcp的租户,全部添加规则,实现本地vlan_id -> 租户全局vxlan_id 的转换,发送到去往各
   个计算节点(compXXX)的隧道口

```
table=21,priority=1,dl_vlan=300,actions=strip_vlan,set_tunnel:8888,output:2,output:3
```

如果还有其他租户的业务,如上所示继续添加流表

_ _ _

#### 租户虚拟机dhcp获取指定ip ####

修改虚拟机comp115-demo0x01的eth0网卡为dhcp

```
[root@localhost ~]# sed -ri "s/(BOOTPROTO=).*/\1dhcp/" /etc/sysconfig/network-scripts/ifcfg-eth0
[root@localhost ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
ONBOOT=yes
NAME=eth0
BOOTPROTO=dhcp
```

查看虚拟机comp115-demo0x01的mac地址

```
[root@comp115 ~]# virsh domiflist comp115-demo0x01
Interface  Type       Source     Model       MAC
-------------------------------------------------------
vnet0      bridge     brg-sg     virtio      52:54:00:22:09:1e
```

修改网络节点dhcp命名空间下的dnsmasq服务的配置文件,将mac绑定预定的ip地址
```
[root@net117 ~]# echo '52:54:00:22:09:1e,150.150.150.10' \
                     > /var/lib/dhcp/d30a8588-f968-4f62-9388-4ed8f61e8355/host
```

通过 dhcp-hostfile 将mac/ip pair写入,可以预期地为指定mac的虚拟机分配指定ip
官方介绍配置文件修改后不需要重启服务,只需要reload即可:

```
The advantage of storing DHCP host information in this file is that 
it can be changed without re-starting dnsmasq the file will be re-read 
when dnsmasq receives SIGHUP
```

找到对应进程pid

```
[root@net117 ~]# cat /var/lib/dhcp/d30a8588-f968-4f62-9388-4ed8f61e8355/pid
19474
[root@net117 ~]# kill -HUP 19474
```

虚拟机comp115-demo0x01重启网络服务,并在查看其ip.

```
[root@localhost ~]# ifdown eth0
[root@localhost ~]# ifup   eth0
Determining IP Information for eth0... done.
```

从 150.150.150.3 登录 150.150.150.10

```
[root@localhost ~]# ssh root@150.150.150.10
root@150.150.150.10's password:
Last login: Thu Dec 29 10:15:54 2016
[root@localhost ~]#
[root@localhost ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1454 qdisc pfifo_fast state UP qlen 1000
    link/ether 52:54:00:22:09:1e brd ff:ff:ff:ff:ff:ff
    inet 150.150.150.10/24 brd 150.150.150.255 scope global dynamic eth0
       valid_lft 85635sec preferred_lft 85635sec
    inet6 fe80::5054:ff:fe22:91e/64 scope link
       valid_lft forever preferred_lft forever
```
_ _ _

#### 租户虚拟机绑定外网IP ####

为虚拟机 150.150.150.3 绑定浮动ip(e.g: 10.160.0.147)

```
[root@net117 ~]# ip netns exec ns-private-router ip addr add dev extgw 10.160.0.147/32 broadcast 10.160.0.147
```

使用iptables进行nat,将虚拟机内网ip与浮动ip进行转换

```
[root@net117 ~]# ip netns exec ns-private-router iptables -t nat -A  PREROUTING  \
                           -d 10.160.0.147 -j DNAT --to-destination 150.150.150.3
[root@net117 ~]# ip netns exec ns-private-router iptables -t nat -A  OUTPUT  \
                           -d 10.160.0.147 -j DNAT --to-destination 150.150.150.3
[root@net117 ~]# ip netns exec ns-private-router iptables -t nat -A  POSTROUTING  \
                           -s 150.150.150.3 -j SNAT --to-source 10.160.0.147
```

注意开启虚拟路由下的路由转发(iptables nat + ip forward 很重要的2项配置)

```
[root@net117 ~]# ip netns exec ns-private-router sysctl -w net.ipv4.ip_forward=1
```

使用浮动ip测试ssh连接10.160.0.147:

```
[root@dev ~]# ssh root@10.160.0.147
The authenticity of host '10.160.0.147 (10.160.0.147)' can't be established.
ECDSA key fingerprint is d2:50:63:f3:82:05:37:6e:17:3e:bc:de:d6:26:16:38.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.160.0.147' (ECDSA) to the list of known hosts.
root@10.160.0.147's password:
Last login: Wed Dec 28 15:56:12 2016
[root@localhost ~]#
```

_ _ _

#### 虚拟机web业务负载均衡 ####

在租户客户机comp115-demo0x01 与 comp114-demo0x01上安装httpd服务,设置简单的首页,并启动服务

```
[root@localhost ~]# rpm -ivh mailcap-2.1.41-2.el7.noarch.rpm
[root@localhost ~]# rpm -ivh apr-1.4.8-3.el7.x86_64.rpm
[root@localhost ~]# rpm -ivh apr-util-1.5.2-6.el7.x86_64.rpm
[root@localhost ~]# rpm -ivh httpd-tools-2.4.6-40.el7.centos.4.x86_64.rpm
[root@localhost ~]# rpm -ivh httpd-2.4.6-40.el7.centos.4.x86_64.rpm
[root@localhost ~]# echo "Greetings from server comp114-demo0x01(150.150.150.3)" > /var/www/html/index.html
[root@localhost ~]# systemctl start httpd
[root@localhost ~]# iptables -I INPUT -p tcp --dport 80 -j ACCEPT
```

在br-int上创建haproxy前端网络设备ha0x01-if,设置本地vlan:tag=300,标识该服务属于demo租户

```
[root@net117 ~]# ovs-vsctl -- --if-exists del-port ha0x01-if -- add-port br-int ha0x01-if \
-- set Interface ha0x01-if type=internal \
-- set Interface ha0x01-if external-ids:iface-id=903431fe-567a-447f-d59e-5fb94aa36a40 \
-- set Interface ha0x01-if external-ids:iface-status=active \
-- set Interface ha0x01-if external-ids:attached-mac=fa:16:00:4c:88:16 \
-- set Port ha0x01-if tag=300
```

在网络节点上创建haproxy命名空间ha0x01,并将ha0x01-if设备放入命名空间下,配置内网前端ip(150.150.150.7)

```
[root@net117 ~]# ip netns add ha0x01
[root@net117 ~]# ip link set ha0x01-if netns ha0x01
[root@net117 ~]# ip netns exec ha0x01 ip link set ha0x01-if mtu 1450 up
[root@net117 ~]# ip netns exec ha0x01 ip link set lo up
[root@net117 ~]# ip netns exec ha0x01 ip addr add 150.150.150.7/24 dev ha0x01-if
[root@net117 ~]# ip netns exec ha0x01 ip route add default via 150.150.150.1 dev ha0x01-if
```

在haproxy命名空间下启动haproxy服务

```
[root@net117 ~]# yum install -y haproxy
[root@net117 ~]# mkdir -p /var/lib/netns/ha0x01-vxlan8888-web/
[root@net117 ~]# cat > /var/lib/netns/ha0x01-vxlan8888-web/haproxy.cfg  << EOF
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
  bind 150.150.150.7:80
  mode http
  default_backend 8f09dbaf-0c78-48f1-8b80-662e75036485
  maxconn 100
  option forwardfor
backend 8f09dbaf-0c78-48f1-8b80-662e75036485
  mode http
  balance leastconn
  option forwardfor
  server 0c7d3741-df7b-4bf3-9396-14cc3c05b5a5 150.150.150.3:80 weight 100
  server 44c18162-301e-42f3-9f0a-5522f426412f 150.150.150.10:80 weight 100
EOF
```
```
[root@net117 ~]# ip netns exec ha0x01 haproxy -f /var/lib/netns/ha0x01-vxlan8888-web/haproxy.cfg
```

在虚拟路由中为内网前端ip绑定外网浮动ip(10.160.0.150),对外提供服务

```
[root@net117 ~]# ip netns exec ns-private-router ip addr add dev extgw 10.160.0.150/32 broadcast 10.160.0.150

[root@net117 ~]# ip netns exec ns-private-router iptables -t nat -A  PREROUTING \
-d 10.160.0.150 -j DNAT --to-destination 150.150.150.7

[root@net117 ~]# ip netns exec ns-private-router iptables -t nat -A  OUTPUT \
-d 10.160.0.150 -j DNAT --to-destination 150.150.150.7

[root@net117 ~]# ip netns exec ns-private-router iptables -t nat -A  POSTROUTING \
-s 150.150.150.7 -j SNAT --to-source 10.160.0.150
```

测试对外提供的web服务

```
[root@net117 ~]# curl 10.160.0.150
Greetings from server comp114-demo0x01(150.150.150.3)

[root@net117 ~]# curl 10.160.0.150

Greetings from server comp115-demo0x01(150.150.150.10)
```

_ _ _

#### 附录 ####

[1] [网络节点 + N计算节点网络结构详细说明图](https://github.com/guojy8993/blogs/blob/master/icehouse-neutron-details.jpg)

[2] neutron网络节点虚拟路由下的iptables规则
```
POSTROUTING
   --> neutron-postrouting-bottom
        |
        |
        |
        |--> neutron-l3-agent-snat
               |
               |
               |--> neutron-l3-agent-float-snat
                      |
                      |---- -s 172.10.6.10/32 -j SNAT --to-source 121.23.12.37


iptables -t nat -A  POSTROUTING  -s 172.172.172.3/32 \
                                 -j SNAT --to-source 10.160.0.147
```

```
OUTPUT
--> neutron-l3-agent-OUTPUT
         |
         |
         |--> -d 121.23.12.37/32 -j DNAT --to-destination 172.10.6.10


iptables -t nat -A  OUTPUT  -d 10.160.0.147/32 \
                            -j DNAT --to-destination 172.172.172.3
```

```
PREROUTING --> neutron-l3-agent-PREROUTING
                 |
                 |
                 |--> -d 121.23.12.37/32 -j DNAT --to-destination 172.10.6.10


iptables -t nat -A  PREROUTING  -d 10.160.0.147/32 \
                                -j DNAT --to-destination 172.172.172.3
```

[3] [neutron vxlan网络br-tun流表逻辑](
	https://github.com/guojy8993/blogs/blob/master/neutron-ovs-flows.jpg)
	
[4] [dhcp request/ack包抓包与解析:arp request](https://github.com/guojy8993/blogs/blob/master/dhcp-request.png)

[5] [dhcp request/ack包抓包与解析:arp ack](https://github.com/guojy8993/blogs/blob/master/dhcp-ack-01.png)

[6] [dhcp request/ack包抓包与解析:arp options](https://github.com/guojy8993/blogs/blob/master/dhcp-ack-opts-details.png)
