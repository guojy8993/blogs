### 目录 ###
1. Docker容器的创建与运行

2. Docker容器的管理

3. Docker容器的暂停/恢复/重启/停止/启动/删除

___

#### 第一部分 Docker 容器的创建与运行 ####
```
[root@docker ~]# docker create --help
Usage:	docker create [OPTIONS] IMAGE [COMMAND] [ARG...]
Create a new container
  -a, --attach=[]                 Attach to STDIN, STDOUT or STDERR                 # 附件到stdin/out/err      
  --add-host=[]                   Add a custom host-to-IP mapping (host:ip)         # 猜测: /etc/hosts
  --blkio-weight                  Block IO (relative weight), between 10 and 1000   # 块设备分配io时的权重
  --blkio-weight-device=[]        Block IO weight (relative device weight)
  --cpu-shares                    CPU shares (relative weight)
  --cap-add=[]                    Add Linux capabilities                       # 使用linux特性
  --cap-drop=[]                   Drop Linux capabilities                      # 不使用linux特性
  --cgroup-parent                 Optional parent cgroup for the container     # 该容器可选的父cgroup(猜测:类似套餐)
  --cidfile                       Write the container ID to the file           # 将容器id写入文件
  --cpu-period                    Limit CPU CFS (Completely Fair Scheduler) period  # 百度: CFS
  --cpu-quota                     Limit CPU CFS (Completely Fair Scheduler) quota   
  --cpuset-cpus                   CPUs in which to allow execution (0-3, 0,1)       # CPU绑定
  --cpuset-mems                   MEMs in which to allow execution (0-3, 0,1)
  --device=[]                     Add a host device to the container    # 添加宿主设备(网络设备?块设备?)到容器
  --device-read-bps=[]            Limit read rate (bytes per second) from a device  # 限制设备每秒读字节数
  --device-read-iops=[]           Limit read rate (IO per second) from a device     # 限制设备每秒读IO数
  --device-write-bps=[]           Limit write rate (bytes per second) to a device   # 限制设备每秒写IO数
  --device-write-iops=[]          Limit write rate (IO per second) to a device      # 限制设备每秒写IO数
  --disable-content-trust=true    Skip image verification                  # 启动容器是否验证镜像
  --dns=[]                        Set custom DNS servers                   # 设置 DNS 服务器("8.8.8.8"..)
  --dns-opt=[]                    Set DNS options                          # 设置 DNS 选项
  --dns-search=[]                 Set custom DNS search domains            # 设置 DNS 查找主机("bar.foo.com")
  -e, --env=[]                    Set environment variables                # 设置环境变量
  --entrypoint                    Overwrite the default ENTRYPOINT of the image     # 重写镜像默认的 EntryPoint
  --env-file=[]                   Read in a file of environment variables     # 从文件导入环境变量
  --expose=[]                     Expose a port or a range of ports           # 对外暴露一个/组端口
  --group-add=[]                  Add additional groups to join               # 加入到N个指定用户(??)中
  -h, --hostname                  Container host name                         # 指定 /etc/hostname
  -i, --interactive               Keep STDIN open even if not attached        # 保持stdin开放(即使尚未attach)
  --ip                            Container IPv4 address (e.g. 172.30.100.104)# 配置指定IPv4
  --ip6                           Container IPv6 address (e.g. 2001:db8::33)  # 配置指定IPv6
  --ipc                           IPC namespace to use                   # 进行跨进程通信时使用的命名空间
  --isolation                     Container isolation level              # 容器隔离水平(Low?Normal?High?)
  --kernel-memory                 Kernel memory limit                    # 内核内存限制(防止容器受攻击拖累宿主)
  -l, --label=[]                  Set meta data on a container           # 设置容器的元数据(??)
  --label-file=[]                 Read in a line delimited file of labels       # 从文件导入容器的元数据
  --link=[]                       Add link to another container                 # 添加链路连接另一个容器
  --log-driver                    Logging driver for container                  # 容器使用的 log driver
  --log-opt=[]                    Log driver options                            # 容器使用的 log driver 选项
  -m, --memory                    Memory limit                                  # 限制内存使用
  --mac-address                   Container MAC address (e.g. 92:d0:c6:0a:29:33)    # 容器网卡的mac(?只能单网卡?)
  --memory-reservation            Memory soft limit                                 # 内存使用的软限制
  --memory-swap                   Swap limit equal to memory plus swap: '-1' to enable unlimited swap # 设置 swap 
  --memory-swappiness=-1          Tune container memory swappiness (0 to 100)# 调整容器swap/(swap+mempory)的占比?
  --name                          Assign a name to the container             #自定义容器名
  --net=default                   Connect a container to a network           #连接容器到指定网络
  --net-alias=[]                  Add network-scoped alias for the container #网络的别名
  --oom-kill-disable              Disable OOM Killer                         #关闭 "容器内存溢出则终止" 的功能
  --oom-score-adj                 Tune host's OOM preferences (-1000 to 1000)#调整宿主对容器OOM的偏好
  -P, --publish-all               Publish all exposed ports to random ports  #发布容器的暴露端口(s)为宿主随机端口(s)
  -p, --publish=[]                Publish a container's port(s) to the host  #发布容器的指定暴露端口为宿主某指定端口
  --pid                           PID namespace to use                              # PID 命名空间(host)
  --privileged                    Give extended privileges to this container        # 给予容器扩展特权(??)
  --read-only                     Mount the container's root filesystem as read only# 以ro方式挂载容器root文件系统
  --restart=no                    Restart policy to apply when a container exits  # 创建是发现已存在容器是否重启之
  --security-opt=[]               Security Options                            # 安全选项(??)
  --shm-size                      Size of /dev/shm, default value is 64MB     # 共享内存(?作用?)大小
  --stop-signal=SIGTERM           Signal to stop a container, SIGTERM by default     # 容器接受何种信号以结束运行
  --sysctl=map[]                  Sysctl options                                     # sysctl 配置
  -t, --tty                       Allocate a pseudo-TTY                              # 分配 pseudo-TTY
  --tmpfs=[]                      Mount a tmpfs directory                            # 挂载 tmpfs 目录
  -u, --user                      Username or UID (format: <name|uid>[:<group|gid>]) # 属主name[:group]/uid[:gid]
  --ulimit=[]                     Ulimit options                                         # Ulimit(??)配置
  --uts                           UTS namespace to use                                   # UTS 命名空间
  -v, --volume=[]                 Bind mount a volume                                    # 挂载卷到容器
  --volume-driver                 Optional volume driver for the container       # 容器的卷驱动(virtio?)
  --volumes-from=[]               Mount volumes from the specified container(s)  # 挂载指定容器作为卷(s)
  -w, --workdir                   Working directory inside the container # 容器工作目录(容器创建后即进入该目录)
```

(1) --add-host 的使用
```
[root@docker ~]# docker run -it --name docker 
                 --add-host www.dockerio.com:172.172.172.172  \
                 --add-host www.dockerhub.com:172.172.172.173 \
                 -v /data/docker:/data 10.160.0.153:5000/centos7:latest /bin/bash
[root@21b652ade8f5 /]# cat /etc/hosts
...
172.172.172.172 www.dockerio.com
172.172.172.173 www.dockerhub.com
172.17.0.2  21b652ade8f5
```

(2) --volume 的使用
```
[root@docker ~]# echo "Vloume of container" > /opt/docker/readme.txt
[root@docker ~]# docker run -it --name docker 
                -v /opt/docker:/data 10.160.0.153:5000/centos7:latest /bin/bash
[root@058bed3f1e17 /]# ls /data/
readme.txt
[root@058bed3f1e17 /]# cat /data/readme.txt 
Vloume of container
```
(3) --workdir 的使用
```
[root@docker ~]# docker run -it --name docker --workdir /data 10.160.0.153:5000/centos7:latest /bin/bash
[root@63286b5ab9b9 data]#
```

(4) --volumes-from 的使用
```
[root@docker ~]# docker run -it --name datashare -v /datashare  10.160.0.153:5000/centos7:latest /bin/bash
[root@7cbc351e0e22 /]# echo datashare > /datashare/readme.txt
[root@docker ~]# docker run -it --name client --volumes-from datashare 10.160.0.153:5000/centos7:latest /bin/bash
[root@633a5b9d25f8 /]# cat /datashare/readme.txt 
datashare
```
> **NOTE:**

> 注意定义 container 名字与共享数据目录名称一致

(5) --uts (Unix time-sharing System,允许容器拥有独立的命名空间与主机名,默认选项)

(6) --ulimit (ulimit 命令,系统调优)
```
[root@docker ~]# docker run -it --rm --name ulmt \
                 --ulimit nofile=1050:1052 10.160.0.153:5000/centos7:base /bin/bash -c 'ulimit -n'
1050
```
> **NOTE:**

> ulimit参数有哪些是个迷!

> nproc -> max user processes

(7) --sysctl (sysctl 命令,调整内核参数)
```
[root@docker ~]# docker run -it --rm --name ulmt --sysctl net.ipv4.ip_forward=1 \
10.160.0.153:5000/centos7:base /bin/bash
[root@8f7ba86d797e /]# sysctl -a | grep ip_forward
net.ipv4.ip_forward = 1
net.ipv4.ip_forward_use_pmtu = 0
```
> **NOTE:**

> net.ipv4.ip_forward=1 容器做路由器时用到

> 更多参数使用 sysctl -a 查看

(8) --user 指定容器的用户[:组]归属
```
[root@docker ~]# docker run -it --name docker --user root:root 10.160.0.153:5000/centos7:base /bin/bash
```

(9) --tmpfs 的用法:把容器的指定目录分配大小与权限挂载为tmpfs
```
[root@docker ~]# docker run -it --name docker --tmpfs /t2mp:rw,size=10m,mode=1777 \
10.160.0.153:5000/centos7:base /bin/bash
[root@78ae8b0a36b0 /]# df | grep t2mp
tmpfs     10240       0     10240   0% /t2mp
```
> **NOTE:**

> 支持的选项参考: man docker-create 查看明细

> [什么是tmpfs?](http://www.tuicool.com/articles/nqQVFjZ)

> [tmpfs的管理?](http://www.linuxidc.com/Linux/2013-12/93747.htm)

(10) --tty 容器分配pseudo-TTY设备并挂载到容器stdin上(,所以在交互模式下docker可以接受输入)
```
[root@docker ~]# docker run -i --tty --name docker 10.160.0.153:5000/centos7:base /bin/bash
[root@8405e7ddaf06 /]#
```
(11) --stop-signal 何种信号触发docker容器的stop事件(使用 kill -l 查看所有信号量)

(12) --shm-size 共享内存分配大小,默认64m
```
[root@docker ~]# docker run -it --name docker --shm-size=1m 10.160.0.153:5000/centos7:base /bin/bash
[root@6b21304e9a69 /]# df | grep shm
shm 1024  0 1024   0% /dev/shm
```
> **NOTE:**
> [/dev/shm是什么以及如何管理?](http://www.linuxidc.com/Linux/2014-05/101818.htm)

(13) --security-opt 安全选项
```
[root@docker ~]# docker run -it --name docker --security-opt seccomp:unconfined \
10.160.0.153:5000/centos7:base /bin/bash
[root@c0774328406f /]#
```
> **NOTE:**

> docker需要第三方的安全组件(seccomp),但是此时是为配置的(unconfined)

> 除非有安全组件才涉及到该配置

(14)  --restart=no 创建容器是遇到已有容器的重启策略(man docker-create 查看具体支持哪些选项)

(15) --read-only 以ro方式挂载容器的root文件系统
```
[root@docker ~]# docker run -it --name docker 10.160.0.153:5000/centos7:base /bin/bash
[root@6813b2798636 /]# touch /root/i-can-write
...
[root@docker ~]# docker run -it --name docker-ro --read-only 10.160.0.153:5000/centos7:base /bin/bash
[root@96a6931c60a9 /]# touch /root/can-i-write
touch: cannot touch '/root/can-i-write': Read-only file system
...
```

(15) --privileged 是否给予容器以扩展特权(true|false),参考默认选项予以禁止
```
[root@docker ~]# docker run -it --name docker-ro \
                  --privileged=false \
                  --tmpfs /t2mp:rw,size=10m,mode=17777 10.160.0.153:5000/centos7:base /bin/bash
[root@208d9f4971c5 /]# umount /t2mp 
umount: /t2mp: must be superuser to umount
...
[root@docker ~]# docker run -it --name docker-privilege \
                  --privileged=true \
                  --tmpfs /t2mp:rw,size=10m,mode=17777 10.160.0.153:5000/centos7:base /bin/bash
[root@3e47c98bf530 /]# umount /t2mp
```
(16) --pid 设置容器的PID Mode,目前仅host选项,且不安全(man docker-create 了解详细)

(17) --publish 对外发布端口
```
[root@docker ~]# docker run -it --name docker --publish 10.160.0.154:8022:22 \
10.160.0.153:5000/centos7:base /bin/bash
[root@a542190b4ff8 /]#
```
> **NOTE:**
> 注:将本地主机 10.160.0.154 的 8022 端口映射为 容器的22 端口

(18) --publish-all 全部容器端口全部对外发布为宿主随机端口(布尔值,true|false,默认为后者)

(19) --oom-score-adj 调整宿主对容器OOM的偏好(数值,-1000~1000)

(20) --oom-kill-disable 关闭 "容器内存溢出则终止" 的功能

(21) --network-alias 用途不明

(22) --net=default 指定连接的网络的模式(详细参考man手册,使用none可以自定义网络)

(23) --name  指定容器名字

(24) --memory-swappiness 调整容器swap(对?(swap+mempory)?)的占比

(25) --memory "<int><k,m,g...>"  指定memory多大

(26) --memory-swap  "<int><k,m,g...>" 指定memory + swap总计多大(与 --memory结合使用)

(27) --memory-reservation 内存使用的软限制
> **NOTE:**

> [详细了解swap,memory(很好的文档!!!)](http://www.cnblogs.com/xuxinkun/p/5541894.html) 

(28) --mac-address  容器网卡的mac
```
[root@docker ~]# docker run -it --name docker --mac-address 02:42:ac:11:ff:ff \
10.160.0.153:5000/centos7:base /bin/bash
[root@c120a92099ec /]# ip -o link show eth0 | awk '{print $14}'
02:42:ac:11:ff:ff
```

(29) --log-driver 容器使用的log driver (使用man查看详细.docker支持 json-file/journald)

(30) --log-opt 容器使用的 log driver 选项
> **NOTE:**
> [journald日志驱动以及驱动选项](https://www.freedesktop.org/software/systemd/man/journald.conf.html)

(31) --link 连接另外一个容器
```
[root@docker ~]# docker run -it --name docker-server 10.160.0.153:5000/centos7:base /bin/bash
[root@89a28267ea37 /]# ip -o addr | grep eth0 | grep "inet " | awk '{print $4}'
172.17.0.2/16
[root@docker ~]# docker run -it --name docker-client --link docker-server 10.160.0.153:5000/centos7:base /bin/bash
[root@10d9c1bb5daf /]# ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.056 ms
...
```

(32) --label 设置容器的元数据

(33) --label-file 从文件导入容器的元数据
```
[root@docker ~]# docker run -it --name docker --label net_eth0_ipaddr=172.17.0.2 \
10.160.0.153:5000/centos7:base /bin/bash
[root@a694fa059fb8 /]# exit
exit
[root@docker ~]# docker inspect -f '{{.Config.Labels.net_eth0_ipaddr}}' a694fa059fb8
172.17.0.2
[root@docker ~]# docker stop a694fa059fb8
a694fa059fb8
[root@docker ~]# docker inspect -f '{{.Config.Labels.net_eth0_ipaddr}}' a694fa059fb8
172.17.0.2
```
> **NOTE:**

> 这个属性有很重要的用途(记录容器一些易丢失的信息,比如网络信息.它是不随容器重启而消失的)

(34) --kernel-memory 参数格式: <int><k,m,g...>
```
[root@docker ~]# docker run -it --name docker --kernel-memory 100m \
10.160.0.153:5000/centos7:base /bin/bash
```
(35) --isolation 执指定使用何种容器隔离技术(默认default)

(36) --ipc 选项有host(不安全)与"container:<容器名|ID>" 两种.

(37) --ip6 与 --net 结合使用(如果使用自定义overlay网络,用不到该项目)

(38) --ip 与 --net 结合使用(如果使用自定义overlay网络,用不到该项目)

(39) --interactive  # 保持stdin开放(即使尚未attach)

(40) --hostname 指定 hostname
```
[root@docker ~]# docker run -it --name docker --hostname=www.huayu.com \
10.160.0.153:5000/centos7:base /bin/bash
[root@www /]# hostname
www.huayu.com
```
(41) --group-add 加入到指定用户组
```
[root@docker ~]# docker run -it --name docker --group-add daemon 10.160.0.153:5000/centos7:base /bin/bash
```
(42) --expose 容器对外暴露端口或端口范围,但并不通过宿主端口映射对外发布(相当于iptables放开端口范围访问许可)

> **NOTE:**

> iptables -I INPUT -p tcp --dport 8000:9000 -j ACCEPT

> 在禁用容器iptables的情况下使用 --expose 选项放开port访问许可

(43) --env-file 从宿主某文件为容器导入环境变量

(44) --env 设置容器环境变量
```
[root@docker ~]# cat > /root/environ << EOF
> user=root
> passwd=it-is-a-secret
> EOF
[root@docker ~]# docker run -it --name docker --env  system=centos7
                 --env-file /root/environ 10.160.0.153:5000/centos7:base /bin/bash
[root@2bdc097c45fa /]# echo ${passwd}
it-is-a-secret
[root@2bdc097c45fa /]# echo ${system}
centos7
```
(45) --entrypoint 覆盖镜像默认的 entrypoint
> **NOTE:**

> [官方说明](https://www.ctl.io/developers/blog/post/dockerfile-entrypoint-vs-cmd/)

(46) --dns 设置 DNS 服务器("8.8.8.8"..)

(47) --dns-opt 设置 DNS 选项

(48) --dns-search 设置 DNS 查找主机("bar.foo.com")
```
[root@docker ~]# docker run -it --name docker \
                            --dns-opt port=55555 --dns 8.8.4.4 \
                            --dns-search bar.foo.com \
                            10.160.0.153:5000/centos7:base /bin/bash   
[root@46cd1e8e1544 /]# cat /etc/resolv.conf 
search bar.foo.com
nameserver 8.8.4.4
options port=55555
```
> **NOTE:**

> [DNS options ?](http://blog.csdn.net/bpingchang/article/details/38427113)

(49) --device=[] 不必使用--privileged即可直接使用宿主的设备(块设备,/de/zero,音频设备)
```
[root@docker ~]# docker run -it --device=/dev/sdb:/dev/xvdb \
                            --device=/dev/zero:/dev/nulo \
                            10.160.0.153:5000/centos7:base ls -l /dev/{xvdb,nulo}
crw-rw-rw-. 1 root root 1,  5 Oct 23 08:09 /dev/nulo
brw-rw----. 1 root disk 8, 16 Oct 23 08:09 /dev/xvdb
```
(50) --device-read-bps=[]            限制块设备每秒读字节数
```
[root@docker ~]# docker run -it \
--device-read-bps=/dev/sda:1mb 10.160.0.153:5000/centos7:base
```
(51) --device-read-iops=[]           限制块设备每秒读IO数
```
[root@docker ~]# docker run -it \
--device-read-iops=/dev/sda:1 10.160.0.153:5000/centos7:base
```
(52) --device-write-bps=[]           限制块设备每秒写字节数
```
[root@docker ~]# docker run -it \
--device-write-bps=/dev/sdb:1mb 10.160.0.153:5000/centos7:base
```
(53) --device-write-iops=[]          限制块设备每秒写IO数
```
[root@docker ~]# docker run -it \
--device-write-iops=/dev/sda:1 10.160.0.153:5000/centos7:base
```
(54) --disable-content-trust=true    启动容器是否验证镜像

(55) --attach=[]                     附加stdin/out/err到容器    

(56) --blkio-weight                  块设备分配io时的权重(10~1000)

(57) --blkio-weight-device=[]        Block IO weight (relative device weight)

(58) --cpu-shares                    CPU shares (relative weight)

(59) --cap-add=[]                    使用linux特性

(60) --cap-drop=[]                   不使用linux特性

(61) --cgroup-parent                 该容器可选的父cgroup(猜测:类似套餐)

(62) --cidfile                       将容器id写入文件
```
[root@docker ~]# docker run -it --name docker --cidfile /root/docker 10.160.0.153:5000/centos7:base /bin/bash
[root@docker ~]# cat /root/docker 
f2110250523ca12c6302168fbcb8c86af860ba33525f656d7f9963c360fde109
```
(63) --cpu-period                    Limit CPU CFS (Completely Fair Scheduler) period  # 百度: CFS

(64) --cpu-quota                     Limit CPU CFS (Completely Fair Scheduler) quota   

(65) --cpuset-cpus                   CPUs in which to allow execution (0-3, 0,1)       # CPU绑定
```
[root@docker ~]# cat /proc/cpuinfo | egrep "^processor\s+:\s+[0-9]+" | wc -l
4
[root@docker ~]# docker run -it --name docker --cpuset-cpus 0-4 10.160.0.153:5000/centos7:base /bin/bash
docker: Error response from daemon: Requested CPUs are not available - requested 0-4, available: 0-3
[root@docker ~]# docker run -it --name docker --cpuset-cpus 0-3 10.160.0.153:5000/centos7:base /bin/bash
[root@15f0fffb8f9a /]#
```
(66) --cpuset-mems  MEMs in which to allow execution (0-3, 0,1)
```
[root@docker ~]# docker run -it --name docker --cpuset-mems 0-3 10.160.0.153:5000/centos7:base /bin/bash
docker: Error response from daemon: Requested memory nodes are not available - requested 0-3, available: 0
[root@docker ~]# docker run -it --name docker --cpuset-mems 0 10.160.0.153:5000/centos7:base /bin/bash
[root@2c40e276b0c7 /]#
```

## 第二部分 Docker容器的管理 ##
1. Docker容器数据管理
1.1 数据卷
1.1.1 在容器内创建一个数据卷
[root@docker ~]# docker run -it --name web -P -v /webapp 10.160.0.153:5000/centos7:base /bin/bash
[root@f443f6ad3bf8 /]# echo "data volumes within container" > /webapp/readme

1.1.2 挂载主机目录作为数据卷
[root@docker ~]# docker run -idt --name web -P -v /data:/webapp:rw 10.160.0.153:5000/centos7:base /bin/bash
e9a9f1e0d89bc015526673282e427652bc5523d4a2607f7ecdd10e8c55ba2a7e
[root@docker ~]# docker inspect -f '{{.State.Pid}}' web
12013
[root@docker ~]# nsenter --target 12013 --pid --mount --ipc --net
[root@docker /]# echo "data volume from host FS" > /webapp/readme.txt
注意: 如果要使用宿主的文件系统需要宿主显式设置 setenforce 0

1.1.3 挂载本地主机文件为数据卷
[root@docker ~]# docker run -idt --name web -P \
                            -v /data/docker-bash.history:/.bash_history 10.160.0.153:5000/centos7:base /bin/bash
27b11696ea8dd45be5cce874a8eb393f35e3450b00aa2f408ac18dbdb787d402

1.2 数据卷容器
[root@docker ~]# docker run -idt --name dbdata -v /dbdata 10.160.0.153:5000/centos7:base /bin/bash
7e6f75faf6623754ece7431616cd8262bd6950eb2700fe608f5a6058cb970d0c
[root@docker ~]# docker run -it --volumes-from dbdata --name db01  10.160.0.153:5000/centos7:base /bin/bash
[root@f2649a03e0f1 /]# echo "db01" > /dbdata/readme.txt
...
[root@docker ~]# docker run -it --volumes-from dbdata --name db02  10.160.0.153:5000/centos7:base /bin/bash
[root@ad51117321e9 /]# echo "db02" >> /dbdata/readme.txt
...
[root@docker ~]# docker inspect -f '{{.State.Pid}}' dbdata
12584
[root@docker ~]# nsenter --target 12584 --pid --mount --net --ipc
[root@docker /]# cat /dbdata/readme.txt 
db01
db02

1.3 使用数据卷容器迁移数据
1.3.1 备份容器数据到本地主机
[root@docker ~]# docker run -it --name dbdata -v /dbdata 10.160.0.153:5000/centos7:base /bin/bash
[root@befa54ef9758 /]# echo "dbdata file" > /dbdata/readme
...
[root@docker ~]# docker run -it --name db01 --volumes-from dbdata \
                            -v /tmp/docker:/backup:rw 10.160.0.153:5000/centos7:base /bin/bash \
                            -c "tar -cf /backup/readme.tar.gz /dbdata/*"
tar: Removing leading `/' from member names
[root@docker ~]# ll /tmp/docker/
...
-rw-r--r--. 1 root root 10240 Oct 23 21:51 readme.tar.gz
# 注意: 如果要使用宿主的文件系统需要宿主显式设置 setenforce 0

1.3.1 从本地主机还原数据到容器
[root@docker ~]# rm -rf /tmp/docker/*
[root@docker ~]# docker rm -f db01
db01
[root@docker ~]# echo "restore files from host to container" > /tmp/docker/readme
[root@docker ~]# docker run -it --name db01 --volumes-from dbdata \
                            -v /tmp/docker:/backup:rw 10.160.0.153:5000/centos7:base /bin/bash \
                            -c "cp -r /backup/* /dbdata/"
[root@docker ~]# docker attach dbdata
[root@befa54ef9758 /]# cat /dbdata/readme
restore files from host to container


# 生产环境下使用 docker devicemaper 存储
# https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/

### Docker容器网络管理 ###
1. 使用linux bridge(默认情况)
[root@docker /]# export REDIS_PORT_6378_TCP=tcp://10.160.0.134:6379
[root@docker /]# env | grep _TCP | sed 's/.*_PORT_\([0-9]*\)_TCP=tcp:\/\/\(.*\):\(.*\)/socat TCP4-LISTEN:\1,fork,reuseaddr TCP4:\2:\3/'
socat TCP4-LISTEN:6378,fork,reuseaddr TCP4:10.160.0.134:6379
'''
待完成
'''
# mariadb 的安装配置:
# http://docs.openstack.org/newton/install-guide-obs/environment-sql-database.html

2. 使用openvswitch(自定义情况)
# 使用--net=none告诉docker不要将新建容器连接到docker0 
[root@docker ~]# docker run -idt --name ovs-demo --net=none 10.160.0.153:5000/centos7:base /bin/bash
6f98f3b0a01fc737082085fb7eb14f4d62afa9be511bf63272c9957865d0d604

# 查看新加容器的pid值
[root@docker ~]# docker inspect -f '{{.State.Pid}}' 6f98f3b0a01fc737082085fb7eb14f4d62afa9be511bf63272c9957865d0d604
20070

# 添加新加容器的netns到ip netns列表
[root@docker ~]# mkdir -p /var/run/netns && ln -s /proc/20070/ns/net /var/run/netns/ovs-demo
[root@docker ~]# ip netns
ovs-demo

# 查看容器的netns中的链路设备,并无eth0
[root@docker ~]# ip netns exec ovs-demo ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
       
# 在宿主上新加ovs网桥连接到宿主的管理网卡
[root@docker ~]# ovs-vsctl add-br br-int
[root@docker ~]# ovs-vsctl add-port br-int ens192

# 在宿主下创建对等设备,一头在br-int上,一头在docker netns里面作为容器的eth0

# 对等设备的up/down状态时刻保持一致.启用一端设备,另一端同步打开.
[root@docker ~]# ip link add ovsdemo type veth peer name eth0 
[root@docker ~]# ovs-vsctl add-port br-int ovsdemo
[root@docker ~]# ip link set eth0 netns ovs-demo
[root@docker ~]# ip link set ovsdemo up

# 给docker配置管理ip,且在ovs对应端口上设置vlan
[root@docker ~]# ip netns exec ovs-demo ip addr add dev eth0 10.160.0.139/16
[root@docker ~]# ip netns exec ovs-demo ip route add default via 10.160.0.1
[root@docker ~]# ovs-vsctl set Port ovsdemo tag=610

# 进入容器测试网络
[root@docker ~]# docker attach ovs-demo
[root@6f98f3b0a01f /]# ssh root@10.160.0.144
root@10.160.0.144's password: 
Last login: Sun Oct 23 21:44:23 2016 from 10.100.0.192
# NOTE:
现在遇到的问题是:重启容器后网卡丢失,需要再次重建.我的解决方法是:将网卡配置写入容器元数据(label),下次启动的
时候解析元数据,执行上述各个步骤.予以修复即可.
思路如下:
(1) 创建容器的时候即把网络信息写入元数据
(2) 后续各次启动的时候解析元数据获取网络配置
(3) 根据网络配置还原容器网络
执行:
1.元数据: 网卡名,网卡ip,cidr,网关,vlan_id
[root@docker ~]# docker run -idt --name docker \
                            --label net_device=eth0 \
                            --label net_eth0_ipaddr=10.160.0.139/16  \
                            --label net_eth0_gateway=10.160.0.1 \
                            --label net_eth0_cidr=10.160.0.0/16 \
                            --label net_eth0_vlan=610  \
                            --net=none 10.160.0.153:5000/centos7:base /bin/bash
634ecaca82ee11096b74355b64b9507455fa7ce6142e72319cb4d00c4ac4fd7d
[root@docker ~]# docker inspect -f '{{.Config.Labels.net_device}}' docker
eth0
[root@docker ~]# docker inspect -f '{{.Config.Labels.net_eth0_ipaddr}}' docker
10.160.0.139/16
[root@docker ~]# docker inspect -f '{{.Config.Labels.net_eth0_gateway}}' docker
10.160.0.1
[root@docker ~]# docker inspect -f '{{.Config.Labels.net_eth0_cidr}}' docker
10.160.0.0/16
[root@docker ~]# docker inspect -f '{{.Config.Labels.net_eth0_vlan}}' docker
610

2.根据上述获取的各个参数,执行前文脚本
[root@docker /]# ip route flush dev eth0
[root@docker /]# ip route add 10.160.0.0/16 dev eth0  proto kernel  scope link  src 10.160.0.139 metric 400
[root@docker /]# ip route add default via 10.160.0.1 dev eth0 proto static  metric 400
# 将上述shell装换为文件 docker-start ,重复使用.
[root@docker ~]# cp docker-start /usr/bin/           
[root@docker ~]# chmod 755 /usr/bin/docker-start


## 第三部分 Docker容器的暂停/恢复/重启/停止/启动/删除 ##
