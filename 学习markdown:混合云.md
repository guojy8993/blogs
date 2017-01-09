### 本文档章节目录 ###
1. Provider网络的初始化
2. 租户网络的初始化
3. 各个计算节点上租户容器与虚拟机的创建
4. 各节点bridge添加fdb entry实现跨宿主租户内网连通
5. 为租户内网IP绑定浮动IP
6. 负载均衡即服务实现
7. 附录

![kvm_vxlan_docker混合云](https://github.com/guojy8993/blogs/blob/master/sys.png)

#### 第一节:Provider网络的初始化 ####


#### 第二节:租户网络的初始化 ####
#### 第三节:各个计算节点上租户容器与虚拟机的创建 ####
#### 第四节:各节点bridge添加fdb entry实现跨宿主租户内网连通 ####
#### 第五节:为租户内网IP绑定浮动IP ####
#### 第六节:负载均衡即服务实现 ####
1. 添加负载均衡前端ip与设备
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
2. 在vxlan网络(某租户网络)内启动负载均衡服务
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
3. 租户内网测试负载均衡服务
```
[root@docker-net127 ~]# ip netns exec private-router curl 192.168.100.8
This Page is got from server 192.168.100.6(nat to 200.160.0.4)
[root@docker-net127 ~]# ip netns exec private-router curl 192.168.100.8
This page is got from 192.168.100.7
```
4. 为负载均衡前端ip绑定浮动ip(200.160.0.5),对公网提供服务
```
[root@docker-net127 ~]# ip netns exec private-router ip addr add 200.160.0.5/32 broadcast 200.160.0.5  dev ss-router-gw
[root@docker-net127 ~]# ip netns exec private-router iptables -t nat -A PREROUTING -d 200.160.0.5 -j DNAT --to-destination 192.168.100.8
[root@docker-net127 ~]# ip netns exec private-router iptables -t nat -A OUTPUT -d 200.160.0.5 -j DNAT --to-destination 192.168.100.8
[root@docker-net127 ~]# ip netns exec private-router iptables -t nat -A POSTROUTING -s 192.168.100.8 -j SNAT --to-source 200.160.0.5
```
5. 公网测试负载均衡服务
```
[root@net ~]# curl 200.160.0.5
This Page is got from server 192.168.100.6(nat to 200.160.0.4)
[root@net ~]# curl 200.160.0.5
This page is got from 192.168.100.7
```

#### 附录 ####
[1] [KVM vxlan docker 混合云详细图](https://github.com/guojy8993/blogs/blob/master/kvm%E4%B8%8Edocker%E6%B7%B7%E5%90%88%E4%BA%91.png)

[2] [计算节点docker容器初始化脚本](https://github.com/guojy8993/blogs/blob/master/Docker%E9%AB%98%E7%BA%A7%E7%BD%91%E7%BB%9C%E9%85%8D%E7%BD%AE%E5%AE%9E%E6%88%98)
