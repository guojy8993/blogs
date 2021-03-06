思路:

在KVM虚拟化环境下,使用iptables+inotify实现自学习防火墙(v1.0):

(1)思路来源:

   在openvswitch(虚拟交换机）中,某流表处理流量,学习traffic的源mac,ip,交换机输入端口,
   并使用learn强制在另一个流表中插入一条规则; 后续流量，优先使用学习表中的规则处理流量; 而iptables
   中的 -j 跳转指定链或或执行指定操作,类似ovs中的actions(对流量的处理),但是前者并没有对应的类似
   learn这样的Operation.此时设想:

a.通过扩展iptables,实现学习防火墙的"学习"操作(较优方案,但实现难度高)

b.使用iptables的 log 模块记录符合"恶意"标准的流量详细信息到系统日志文件,并使用第三方工具实时分析流
  量信息生成防火墙规则(方案具可行性,难度相对低)

(2)方案设计

```
FORWARD ---- > BlackListChain ----> DispatchChain ------> PrivateChain ----> FirewallLogChain --+
                                                                                                |
                                                                                                |
 FORWARD  <-------- BlackListChain  <---------- DispatchChain <---------- PrivateChain <--------+
```

说明:

BlackListChain 全局链,学习链(屏蔽恶意访问者(s)对全体虚拟机的访问).由 inotify负责维护学习防护规则.

DispatchChain  转发链,负责将各个虚拟机的流量导入到虚拟机的私有处理链.

PrivateChain   虚拟机的私有处理链.功能包括检测记录异常访问,业务流量统计.

FirewallLogChain 公共防火墙链.专职匹配不合法流量,并记录; 允许管理员添加规则特定匹配特定特征流量.

实现:

(1)环境搭建:

a.宿主添加管理网络网卡ens224(vlan 610)

```
  [root@my-pc ~]# ip link add link ens224 name ens224.610 type vlan id 610
  [root@my-pc ~]# ip link set ens224.610 up
```

b.添加linux bridge:
```
  [root@my-pc ~]# brctl addbr node02br 
  [root@my-pc ~]# ip link set node02br up

```
c.挂载宿主物理网卡到linux bridge

```
  [root@my-pc ~]# brctl addif ens224.610
```

d.启动虚拟机 node02

```
[root@my-pc ~]# virt-install -n node02 --description node02 \
                --ram 1024 --vcpus 1 --cpu host-model \
                --accelerate --hvm --network bridge:node02br \
                --disk /data/instance/node02/system,bus=virtio,cache=writeback,driver_type=qcow2,size=10 \
                --boot hd,cdrom --graphics vnc,listen=0.0.0.0  \
                --serial file,path=/data/instance/node02/console.log --noautoconsole \
                --input tablet,bus=usb --cdrom /data/iso/welcome.iso

```

e.使用vncview视图进入虚拟机,为虚拟机配置vlan610 的ip,e.g: 10.160.0.151/16

```
[root@localhost ~]# echo "apache-server" > /etc/hostname
[root@apache-server ~]# ip addr add 10.160.0.151/16 dev eth0
[root@apache-server ~]# echo "proxy=http://10.158.89.23:3128" >> /etc/yum.conf
[root@apache-server ~]# yum install -y httpd
[root@apache-server ~]# echo "apache" > /var/www/html/index.html
[root@apache-server ~]# iptables -I INPUT -p tcp --dport 80 -j ACCEPT
[root@apache-server ~]# systemctl start httpd && systemctl enable httpd
```

f.在内网另外机器10.160.0.130上curl访问该服务:

```
[root@network ~]# curl 10.160.0.151
apache
```

g.在宿主上操作iptables规则:

```
[root@my-pc ~]# iptables-save > /data/iptables-backup.old
```

添加BlackListChain，DispatchChain，FirewallLogChain三个公共链,以及node02的私有链

```
[root@my-pc ~]# iptables -N black-list-drop
[root@my-pc ~]# iptables -I FORWARD -j black-list-drop
[root@my-pc ~]# iptables -N traffic-dispatch
[root@my-pc ~]# iptables -N node02-security-meter
[root@my-pc ~]# iptables -N invalid-traffic-logger
```

重要:允许使用 netfilter 管理bridge流量

```
[root@my-pc ~]# sysctl -w net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-iptables = 1
```

为BlackListChain添加默认规则

BlackListChain 处理流量的3个层级

(1)使用inotify维护的DROP规则丢弃恶意源发送的流量

(2)进入DispatchChain

(3)经DispatchChain返回后,再由当前链跳回FORWARD

```
[root@my-pc ~]# iptables -A black-list-drop -j traffic-dispatch
[root@my-pc ~]# iptables -A black-list-drop -j RETURN
```

为DispatchChain添加规则

DispatchChain 处理流量分为2个层级

(1)将流量派发到各个虚拟机的私有链PrivateChain

(2)经PrivateChain处理的流量返回到当前链后再次跳转回BlackListChain链

```
[root@my-pc ~]# iptables -A traffic-dispatch -j RETURN
```

以node02为例添dispatch规则

```
[root@my-pc ~]# virsh domiflist node02
 Interface  Type       Source     Model       MAC
 -------------------------------------------------------
 vnet0      bridge     node02br   rtl8139     52:54:00:9c:b3:24
 [root@my-pc ~]# iptables -I traffic-dispatch -s 10.160.0.151/32 -m physdev \
                          --physdev-in vnet0 --physdev-is-bridged -j node02-security-meter
 [root@my-pc ~]# iptables -I traffic-dispatch -d 10.160.0.151/32 -m physdev \
                          --physdev-out vnet0 --physdev-is-bridged -j node02-security-meter
```

为PrivateChain添加规则

node02-security-meter 的流量处理分为2个层级

(1)将流量重定向到公共匹配链invalid-traffic-logger发掘非法流量的来源

(2)将经由invalid-traffic-logger跳转回来的流量重新跳转回上级链(DispatchChain)

```
[root@my-pc ~]# iptables -A node02-security-meter -j invalid-traffic-logger
[root@my-pc ~]# iptables -A node02-security-meter -j RETURN
```

为FirewallLogChain添加规则

FirewallLogChain 的流量处理分为2个层级

(1)处理各个虚拟机的流量,按管理员添加的匹配规则匹配非法流量,并将(流量)信息记入系统日志

(2)将流量返回各个虚拟机的私有链

```
[root@my-pc ~]# iptables -A invalid-traffic-logger -j RETURN
```   

查看当前链与规则

```
[root@my-pc ~]# iptables -S | egrep \
               "black-list-drop|traffic-dispatch|node02-security-meter|invalid-traffic-logger"
-N black-list-drop
-N invalid-traffic-logger
-N node02-security-meter
-N traffic-dispatch
-A FORWARD -j black-list-drop
-A black-list-drop -j traffic-dispatch
-A black-list-drop -j RETURN
-A invalid-traffic-logger -j RETURN
-A node02-security-meter -j invalid-traffic-logger
-A node02-security-meter -j RETURN
-A traffic-dispatch -d 10.160.0.151/32 -m physdev \
                               --physdev-out vnet0 --physdev-is-bridged -j node02-security-meter
-A traffic-dispatch -s 10.160.0.151/32 -m physdev \
                               --physdev-in vnet0 --physdev-is-bridged -j node02-security-meter
-A traffic-dispatch -j RETURN
```

由管理员添加全局过滤规则

```
[root@my-pc ~]# iptables -I invalid-traffic-logger -p tcp \
                 --syn -m limit --limit 1/s -j LOG \
                 --log-level 5 --log-prefix "EvilTraffic"
```

方案验证:

第三方客户端机器(10.160.0.130,hostname:network)信息如下(注意设备mac地址):

```
[root@network ~]# ip -o a | grep 10.160.0.130
4: ens33.610    inet 10.160.0.130/24 brd 10.160.0.255 scope global ens33.610
                valid_lft forever preferred_lft forever
[root@network ~]# ip -o link | grep ens33.610
4: ens33.610@ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT 
                link/ether 00:0c:29:5e:5d:89 brd ff:ff:ff:ff:ff:ff
```

测试此时网络服务是否可用

```
[root@network ~]# curl 10.160.0.151
apache
```

创建并启动 inotify 脚本,监控/var/log/messages文件,从非法流量记录中学习非法访问者的源地址,并维护宿主 black-list-drop 链的丢弃规则

```
[root@my-pc ~]# cat > iptables_log_monitor << EOF
#! /bin/bash

learn_chain="black-list-drop"
iptlog="/var/log/messages"
iptables=$(which iptables)
keyword="EvilTraffic"

function learn()
{
    mac=$(echo ${1}|awk '{print $10}'|awk -F'=' '{print $2}'|cut -c 19-35)
    cmd="iptables -I ${learn_chain} -m mac --mac-source ${mac} -j DROP"
    chk="iptables -C ${learn_chain} -m mac --mac-source ${mac} -j DROP"
    /usr/bin/sh -c "${chk}" 2>/dev/null
    [[ $? -gt 0 ]] && (/usr/bin/sh -c "${cmd}") 2>/dev/null 
}

inotifywait -mq -e modify ${iptlog} | while read event
do
    line=$(tail -n1 ${iptlog})
    count=$(echo ${line}|grep ${keyword}|wc -l)
    if [ ${count} -gt 0 ];then
        learn "${line}"
    fi
done
EOF

[root@my-pc ~]# nohup sh iptables_log_monitor &
```

观察宿主系统日志输出

```
[root@my-pc ~]# tail -n1 /var/log/messages
Sep 12 02:44:14 my-pc kernel: EvilTrafficIN=node02br OUT=node02br PHYSIN=ens224.610 \
PHYSOUT=vnet0 MAC=52:54:00:9c:b3:24:00:0c:29:5e:5d:89:08:00 SRC=10.160.0.130 DST=10.160.0.151 \
LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=58445 DF PROTO=TCP SPT=55641 DPT=80 WINDOW=29200 RES=0x00 SYN URGP=0
```

观察宿主iptables的black-list-drop规则

```
[root@my-pc ~]# iptables -S | grep "black-list-drop" | grep "mac-source"
-A black-list-drop -m mac --mac-source 52:54:00:9C:B3:24 -j DROP
```
此时可知,规则已经学习到

恶意访问来源10.160.0.130的流量已经在black-list-drop中按mac地址丢弃

测试此时网络服务是否可用

```
[root@network ~]# curl 10.160.0.151 --connect-timeout 10
curl: (28) Connection timed out after 10001 milliseconds
```
    
#### 参考文档: ####

[1] [Linux Bridge](http://bwachter.lart.info/linux/bridges.html)

[2] [使用iptables防御DDOS](https://javapipe.com/iptables-ddos-protection)

[3] [Linux二层防火墙](http://net.doit.wisc.edu/~dwcarder/captivator/linux_l2_firewalling.txt)

[4] [iptables扩展模块](http://ipset.netfilter.org/iptables-extensions.man.html)
    
    
    
    
    
    
    
    
    
    
