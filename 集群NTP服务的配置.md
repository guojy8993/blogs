#### 1.实践环境与设备说明 ####
```
[root@master ~]# cat /etc/hosts | grep openstack
10.160.0.116  master.openstack.org
10.160.0.118  comp118.openstack.org
10.160.0.119  comp119.openstack.org
```

需求:

为该openstack(newton版本)测试环境,需要配置ntp服务实现时间同步,要求 master.openstack.org 使用机房内部另外的时间服

务器(10.150.89.99)校对时间,而集群的其他服务器以master.openstack.org的时间为准进行时间校对.

#### 2.各个节点的软件安装与ntp配置 ####
(1) 针对 *.openstack.org 节点，安装ntp软件: yum install -y ntp

(2) 针对 master.openstack.org 修改 /etc/ntp.conf 配置如下:

```
[root@master ~]# cat /etc/ntp.conf | egrep -v "^#|^$"
driftfile /var/lib/ntp/drift
restrict default nomodify notrap nopeer noquery
restrict 127.0.0.1 
restrict ::1
restrict -4 default kod notrap nomodify
restrict -6 default kod notrap nomodify
server 10.150.89.99 iburst
includefile /etc/ntp/crypto/pw
keys /etc/ntp/keys
disable monitor
```

(3) 针对 comp*.openstack.org 修改 /etc/ntp.conf 配置如下:

```
[root@comp118 ~]# cat /etc/ntp.conf | egrep -v "^#|^$"
driftfile /var/lib/ntp/drift
restrict default nomodify notrap nopeer noquery
restrict 127.0.0.1 
restrict ::1
server master.openstack.org iburst  # 注意该机器应当可以解析master.openstack.org
includefile /etc/ntp/crypto/pw
keys /etc/ntp/keys
disable monitor
```

(4) 针对 *.openstack.org 节点启动ntpd服务,并设置开机启动

```
[root@master ~]# systemctl start ntpd && systemctl enable ntpd && systemctl status ntpd
[root@comp118 ~]# systemctl start ntpd && systemctl enable ntpd && systemctl status ntpd
```

comp118 亦如 comp118操作.

#### 3.测试 ####

(1) 在 master.openstack.org 上验证
```
[root@master ~]# ntpq -c peers
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*10.154.88.88    211.233.40.78    3 u  177 1024  377    0.609  -12.664   5.359

[root@master ~]# ntpq -c assoc

ind assid status  conf reach auth condition  last_event cnt
===========================================================
  1 56203  961a   yes   yes  none  sys.peer    sys_peer  1
```

(2) 在 comp*.openstack.org 上验证

```
[root@comp118 ~]# ntpq -c peers
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*master.openstac 10.154.88.88     4 u    -   64    1    0.280  -18.462   0.101

ind assid status  conf reach auth condition  last_event cnt
===========================================================
  1 63722  963a   yes   yes  none  sys.peer    sys_peer  3
```
