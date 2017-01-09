### 文档内容提要 ###
1.以指定用户身份在指定池中创建数据卷

2.在客户端机器中进行rbd映射,将数据卷映射为本地设备

3.格式化数据卷,挂载,进行读写测试

4.持久化块设备映射信息到本地,设置开机映射

_ _ _

#### 第一部分: 以指定用户身份在指定池中创建数据卷 ####
```
[root@dev ~]# rbd create volumes/base --size 10240 --name client.cinder \
                                      --image-format 2 --image-features +1
[root@dev ~]# rbd ls --pool volumes
base
```
_ _ _

#### 第二部分: 在客户端机器中进行rbd映射,将数据卷映射为本地设备 ####
```
[root@dev ~]# rbd map --image base --name client.cinder --pool volumes
/dev/rbd0
[root@dev ~]# rbd showmapped --name client.cinder
id pool    image snap device
0  volumes base  -    /dev/rbd0
```

#### 第三部分: 格式化数据卷,挂载,进行读写测试 ####
```
[root@dev ~]# mkfs.xfs /dev/rbd0
[root@dev ~]# mkdir -p /mnt/rbd
[root@dev ~]# dd if=/dev/zero of=/mnt/rbd/test count=100 bs=1M
100+0 records in
100+0 records out
104857600 bytes (105 MB) copied, 0.172899 s, 606 MB/s
```

#### 第四部分: 持久化块设备映射信息到本地,设置开机映射 ####
```
[root@dev ~]# wget https://raw.githubusercontent.com/ksingh7/ceph-cookbook/master/rbdmap\
-O /etc/init.d/rbdmap
```

实际操作中发现,rbdmap脚本在hammer版本ceph及其依赖包安装后已经存在

```
[root@dev ~]# echo "volumes/base id=cinder, keyring=$(ceph auth get-or-create client.cinder \
                     | grep key | awk '{print $3}')" >> /etc/ceph/rbdmap

[root@dev ~]# cat /etc/ceph/rbdmap
# RbdDevice		Parameters
volumes/base id=cinder, keyring=AQCbbWxYPM5tBBAARTHNRPzuR5A2t/peKuFmIA==
```

注1: rbd命令选项中的"--name"是以"TYPE.ID"形式出现的,例如:client.cinder表明这是一个ceph客户端,
    且id是cinder; rbdmap中的id由此而来

```
[root@dev ~]# echo "/dev/rbd0 /mnt/rbd xfs defaults,_netdev 0 0 " >> /etc/fstab
[root@dev ~]# /etc/init.d/rbdmap start
[root@dev ~]# echo "/etc/init.d/rbdmap start" >> /etc/rc.local
[root@dev ~]# chmod +x /etc/rc.local
```
