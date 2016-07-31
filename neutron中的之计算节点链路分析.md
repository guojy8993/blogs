计算节点虚拟机实例信息如下:

NAME       NETWORK(fixed_ip,floating_ip)                fixed_ip_mac       device   
hyp-vm1    net-hyp=192.168.66.2          122.114.124.21 fa:16:3e:23:91:7b  tap156d4a84-8f
hyp-vm2    net-hyp=192.168.66.4          -              fa:16:3e:61:d8:b7  tap9b3f4311-42
liyang-vm  net-liyang0993=192.168.100.3  122.114.124.45 fa:16:3e:6c:c9:bb  tap0620d5f8-02

(虚拟机ip与网卡信息可以从 dashboard 以及 计算节点的virsh管理工具 收集,此处略)

[root@compute home]# brctl show
bridge name    bridge id         STP enabled  interfaces
qbr0620d5f8-02 8000.e2ac0a53cb2b no           qvb0620d5f8-02  tap0620d5f8-02
qbr156d4a84-8f 8000.62f2e45acb48 no           qvb156d4a84-8f  tap156d4a84-8f
qbr9b3f4311-42 8000.9ea9c59ddd0b no           qvb9b3f4311-42  tap9b3f4311-42

虚拟机是直连linux bridge的,且每个虚拟机拥有独立的qbr.

qbr是neutron 安全组 具体实现的地方,留待专题说明.

而每个 qbr的 qvb设备 都与 br-int上某对应qvo设备以对等设备的形式直连,参见下文:

[root@compute home]# ovs-vsctl show
1db94f48-8307-4bad-b109-30897062c4e4
    """br-tun 信息略"""
    Bridge br-int
        fail_mode: secure
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port br-int
            Interface br-int
                type: internal
        Port "qvo156d4a84-8f"
            tag: 2
            Interface "qvo156d4a84-8f"
        Port "qvo9b3f4311-42"
            tag: 1
            Interface "qvo9b3f4311-42"
        Port "qvo0620d5f8-02"
            tag: 2
            Interface "qvo0620d5f8-02"
    ovs_version: "2.4.0"

这里举例说明一下: 如何创建对等设备, linux网桥 , 开启并设置对等接口的混杂模式,并设置属主(add-port/addif to ovs/bridge)

ip link add qvb0620d5f8-02 type veth peer name qvob0620d5f8-02
brctl add-br qbr0620d5f8-02
ifconfig qvb0620d5f8-02 promisc up 
ifconfig qvob0620d5f8-02 promisc up
ovs-vsctl add-port  br-int qvob0620d5f8-02 -- set Port qvob0620d5f8-02 tag=2
ovs-vsctl add-port  qbr0620d5f8-02  qvb0620d5f8-02

这样就实现ovs虚拟交换机br-int与linux bridge - qbr 之间的链路互通 .

在linux bridge(安全组网桥),br-int交换机,以及两者之间的链路建立之后我们把注意力放在br-int对 虚拟机(liyang-vm,tap0620d5f8-02网卡的属主)inbound/outbound流量的处理上.

我们已经知道,在br-int上:
Port "qvo0620d5f8-02"
    tag: 2
    Interface "qvo0620d5f8-02"
那么这些信息作何解释? tag=2 是什么? tag 指的是 vlan tag,经此端口的流量,出则mod_vlan_vid,入则 strip_vlan.但是VLAN ID为什么是2?
此处vlan tag 是本地vlan(在当前计算节点),而不是全局的(即整个集群).

liyang-vm 连接的是私有网络 net-liyang0993

[root@controller ~(keystone_admin)]# neutron net-list | grep liyang
| 6a169cca-282d-4bea-a99e-d725ac1721b9|net-liyang0993|1a7ed44b-e61d-4f98-aa47-d9123827acc2 
192.168.100.0/24|

[root@controller ~(keystone_admin)]# neutron net-show 6a169cca-282d-4bea-a99e-d725ac1721b9

+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 6a169cca-282d-4bea-a99e-d725ac1721b9 |
| name                      | net-liyang0993                       |
| provider:network_type     | vxlan                                |
| provider:physical_network |                                      |
| provider:segmentation_id  | 10                                   |
| router:external           | False                                |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   | 1a7ed44b-e61d-4f98-aa47-d9123827acc2 |
| tenant_id                 | c9baab896503480b80919a4f0762dcf8     |
+---------------------------+--------------------------------------+

我们知道该私有网络 VLAN Tag 是根据配置的VNI范围(参考 /etc/neutron/plugin.ini : vni_ranges =10:100) 选取的最小的可用值.
那么我猜想链路中必须有 br-int port 的 vlan-tag 以及 后续的br-tun tunnel_id 以及 provider:segmentation_id 之间的转换处理,留待后续探究.

接上文继续探索 br-int,我们查看 br-int 上的 flow rules:

[root@compute ~]# ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
cookie=0x0, duration=145733.004s, table=0, n_packets=238070, n_bytes=37626893, 
idle_age=0, hard_age=65534, priority=1 actions=NORMAL
cookie=0x0, duration=145732.923s, table=22, n_packets=0, n_bytes=0, idle_age=65534,
hard_age=65534, priority=0 actions=drop

很平常的两条流表规则:
(1) 根据优先级,优先对流量执行Normal转发.
(2) table 的数值不同代表当前规则属于不同的流表, 处理不同的流量 [ 流量归类,按表批量处理 ] ;

然后我们在看看br-int上的patch-tun,br-tun上的patch-int 端口.

ovs虚拟交换机之间的会连是通过 patch-ports 来实现的,下面举例说明 br-int 如何与 br-tun如何实现链路互通.

[root@compute ~]#ovs-vsctl -- --if-exists del-port patch-tun -- add-port br-int \
                 patch-tun -- set Interface patch-tun type=patch options:peer=patch-int
[root@compute ~]#ovs-vsctl -- --if-exists del-port patch-int -- add-port br-tun \
                 patch-int -- set Interface patch-int type=patch options:peer=patch-tun

参考: http://blog.scottlowe.org/2012/11/27/connecting-ovs-bridges-with-patch-ports/
此处两个互添 patch port 的语句将br-int 与 br-tun连接到一起

接着继续往下看 br-tun上的端口与flow rules(参考后文流表规则)，先梳理一下再细说:

首先,br-tun上的ofport信息如下:

[root@compute home]# ovs-ofctl show br-tun
OFPT_FEATURES_REPLY (xid=0x2): dpid:000056bb20081443
n_tables:254, n_buffers:256
capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src 
mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
 1(patch-int): addr:12:cc:14:b8:9a:67
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
 2(vxlan-0aa0006b): addr:42:9e:52:f8:0c:9a
     config:     0
     state:      0
     speed: 0 Mbps now, 0 Mbps max
 LOCAL(br-tun): addr:56:bb:20:08:14:43
     config:     PORT_DOWN
     state:      LINK_DOWN
     speed: 0 Mbps now, 0 Mbps max
OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0

其次 , 需要了解, br-tun流表规则中不同的table 编号表示当前规则所属的表,表示不同的作用.
具体详见源码(neutron/plugins/openvswitch/common/constants.py)说明:

PATCH_LV_TO_TUN = 1
GRE_TUN_TO_LV = 2
VXLAN_TUN_TO_LV = 3
LEARN_FROM_TUN = 10
UCAST_TO_TUN = 20
FLOOD_TO_TUN = 21
CANARY_TABLE = 22

然后我们开动对br-tun上的流表规则的分析.

下面我们逐条分析流表规则:

table=0,priority=1,in_port=1 actions=resubmit(,1)

如果是priority=1,table 0,且从ofport=1的端口进入的流量 重新提交到table 1处理

table=0,priority=1,in_port=2 actions=resubmit(,3)

如果是priority=1,table 0,且从ofport=1 的端口进入的流量 重新提交到table 3处理

table=0,priority=0 actions=drop

为table 0设置默认流表规则:DROP

table=1,priority=1,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)

如果是priority=1的单播(unicast)流量,那么重新提交到 table 20处理.
dl_dst=01:00:00:00:00:00/01:00:00:00:00:00   匹配多播(含广播) Ethernet帧
dl_dst=00:00:00:00:00:00/01:00:00:00:00:00   匹配单播 Ethernet帧

table=1,priority=1,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,21)

如果是广播流量那么重新提交到table 21处理

table=2,priority=0 actions=drop

为table 2设置默认流表规则:DROP

table=3,priority=1,tun_id=0xa actions=mod_vlan_vid:1,resubmit(,10)

如果是从网络节点经vxlan隧道(隧道编号 0xa)传到当前计算节点的流量,将其vlan修改为1，重提给 table 10处理.
特别说明: 此处建立了 tunnel_id 与 本地 vlan间的转换关系即 tunnel 0xa 对应 local vlan(LV) 1.

table=3,priority=1,tun_id=0xb actions=mod_vlan_vid:2,resubmit(,10)

同上

table=3,priority=0 actions=drop

为table 3设置默认流表规则:DROP

table=10,priority=1 actions=learn(table=20,hard_timeout=300,priority=1,NXM_OF_VLAN_TCI[0..11
],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_T
UN_ID[],output:NXM_OF_IN_PORT[]),output:1



此处开篇幅讲解 ovs 的 `learn` action:
ovs-ofctl man手册的说法是 将learn(**) 里面的 ** 作为一条流规则添加/修改
个人看法: 用以模仿物理交换机的学习功能,物理交换机的的经过学习知道对特定特征的流量怎么处理;而对应地ovs
这个虚拟交换机则是一条流表规则(匹配特定特征的流量,并规定一个处理动作) ; 因此 ** 部分参数,至少包含一个匹配规则与一个action ;
learn 中匹配规则说明 示例:
NXM_OF_VLAN_TCI[0..11] 表示匹配 TCI信息的低12位,即VLAN ID域.
NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[] 表示学习当前数据包的mac地址.
output:NXM_OF_IN_PORT[] 表示 新加流表规则的处理出包的操作(将其再次从入口`ovs port`返回)
load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[] 表示新加流表规则 处理出流量时会将其 tunnel id修改为当前流表的 tunnel id.
hard_timeout 表示交换机entry过期时间,一般 learn 必须要加选项,保证定时更新 entry;
如前所示,learn是在table 20(mac学习表)中动态添加flow,因此在table=20的所有flows中,符合格式"table=20,hard_timeout=300, priority=1,vlan_tci={tci}/{mask},dl_dst={mac} actions=load:0->NXM_OF_VLAN_TCI[],load:0xa->NXM_NX_TUN_ID[],output:{port}"的 流表都是通过学习动态添加的流表规则 ; 不符合该格式的流表是br-tun/table-20创建之初添加的初始流表,这是需要注意的地方.

参考:
http://www.tuicool.com/articles/am6n2iB
http://www.cnblogs.com/CasonChan/p/4754486.html
http://www.cnblogs.com/popsuper1982/p/3810271.html
http://blog.csdn.net/gaopeiliang/article/details/45720785
http://m.blog.csdn.net/blog/canxinghen/46761591


table=20,priority=1,vlan_tci=0x0001/0x0fff,dl_dst=fa:16:3e:c3:23:47 
actions=load:0->NXM_OF_VLAN_TCI[],load:0xa->NXM_NX_TUN_ID[],output:2

table=20,priority=1,vlan_tci=0x0002/0x0fff,dl_dst=fa:16:3e:11:3f:0f 
actions=load:0->NXM_OF_VLAN_TCI[],load:0xb->NXM_NX_TUN_ID[],output:2



匹配 table=20,priority=1,vlan id=2 且访问私有网络网关(虚拟路由 qrouter-e0014375-0cb7-44b1-84cb-f7f49dd59344下内网网关 qr-f7f87f9f-54 mac fa:16:3e:11:3f:0f )的流量.

`load` 是什么处理?  load:src[start:end]->dst[start:end] 或 load:value-> dst[start:end]
 作用就是将 load后的第一个参数(->前) 或 参数按(start1:end1)位取出的部分 加载入 第二个参数的start2:end2 (赋值操作);
下面说两个 Ethernet 帧中的 特殊域:
NXM_OF_VLAN_TCI:VLAN帧的tci域,2字节,见下文;此处 load:0->NXM_OF_VLAN_TCI[] 表示清空(priority,cfi,vlan-id),即strip_vlan;
NXM_NX_TUN_ID: Ethernet网帧的tunnel-id域; 此处 load:0xb->NXM_NX_TUN_ID[] 表示设置 tunnel id 为0xb ,跟后文set_tunnel:0xb 一样,个人猜想应该是ovs版本历史变更问题导致的两种形式并存;


此处开篇幅细说 vlan_tci:
首先什么是vlan tci ? (引自 http://blog.csdn.net/kulung/article/details/6442804)
VLAN-TAG 包含2字节的 TPID(标签协议标识) 以及 2字节的 TCI(标签控制信息)
TPID 标识 IEEE 类型,常见的格式是: 802.1Q, 802.{N}{A..Z};
TCI 信息又包含 子域 信息:
 priority :  3bit ,表示0-7 优先级;
 cfi,即 Canonical Format Indicator 占1bit ,0表示规范,1表示不规范;
 vlan id: 占12bit 表示 0~4095,这就是 vlan 最多 4096个的原因 ;

回头看此处 vlan_tci 表示的是 {tci}/{mask} 格式,那这是什么意思呢? ovs-ofctl man 手册解释说: tci与mask 位数
相同(16bit)且 mask的每bit 控制 流表 匹配tci对应bit时使用的的匹配机制:精准匹配或模糊匹配(wildcast,等于*)
那么此处  0x0002/0x0fff 表示精确匹配vlan id=2(0x0fff : 后12位 fff),不限制格式(0x0fff: 中间的一位是0表示不限格式, 0x0fff高3位全0,表示0-7优先级的帧皆可匹配) 的 vlan帧 ;



继续 :

table=20,priority=0 actions=resubmit(,21)

为table 20设置默认流表规则: 重新提交到 table 21处理

table=21,dl_vlan=1 actions=strip_vlan,set_tunnel:0xa,output:2

如果是从 br-int 送入 br-tun的流量 , vlan tag=1 那么去vlan-tag 且设置tunnel_id 0xa 输出到 ofport=2的端

口(即隧道端口 )

table=21,dl_vlan=2 actions=strip_vlan,set_tunnel:0xb,output:2

如果是从 br-int 送入 br-tun的流量 , vlan tag=2 那么去vlan-tag 且设置tunnel_id 0xb 输出到 ofport=2的端

口(即隧道端口)

table=21,priority=0 actions=drop

为table 21设置默认流表规则:DROP

br-tun流表至此算是讲解完了, 那么br-tun 的流表规则表达的是一套怎样的数据处理系统呢? 可以参考下图:



那么此处再讲解 计算节点与网络节点之间vxlan隧道的建立:

[root@compute neutron]# ovs-vsctl -- --if-exists del-port vxlan-0aa0006b -- add-port \
                        br-tun vxlan-0aa0006b -- set Interface vxlan-0aa0006b type=vxlan \ 
                        options:remote_ip=10.160.0.107 options:in_key=flow  \ 
                        options:out_key=flow    options:local_ip=10.160.0.106
[root@network neutron]# ovs-vsctl -- --if-exists del-port vxlan-0aa0006a -- add-port \
                        br-tun vxlan-0aa0006a -- set Interface vxlan-0aa0006a type=vxlan \
                        options:remote_ip=10.160.0.106 options:in_key=flow  \
                        options:out_key=flow    options:local_ip=

其中: compute/10.160.0.106 network/10.160.0.107 皆为相关节点的管理ip.
