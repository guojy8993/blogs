### 本文档内容提要 ###
1.部署ceph块设备客户端

2.配置virsh secret

3.在指定存储池中创建块设备

4.组装kvm虚拟机可用的块设备配置文件

5.以xml形式添加块设备到kvm虚拟机

6.测试磁盘读写

_ _ _

### 第一部分: 部署ceph块设备客户端 ###

参考文档:Ceph块设备客户端的部署(略)

_ _ _

### 第二部分: 配置virsh secret ###

step1: 生成密钥uuid(建议一次生成,各计算节点统一使用)
```
[root@dev ~]# uuidgen
1fb2b027-08d6-4739-95d1-e4f7aca367ba
```

step2: 编辑生成virsh secret文件
```
[root@dev ~]# cat > secret.xml << EOF
<secret ephemeral='no' private='no'>
    <uuid>1fb2b027-08d6-4739-95d1-e4f7aca367ba</uuid>
    <usage type='ceph'>
        <name>client.cinder secret</name>
    </usage>
</secret>
EOF
```

step3: 按xml文件形式创建secret

```
[root@dev ~]# virsh secret-define --file secret.xml
Secret 1fb2b027-08d6-4739-95d1-e4f7aca367ba created
```

step4: 给secret赋值:客户端keyring的key的base64加密字符串

```
[root@dev ~]# virsh secret-set-value --secret 1fb2b027-08d6-4739-95d1-e4f7aca367ba \
    --base64 $(cat /etc/ceph/ceph.client.cinder.keyring | grep key | awk '{print $3}') && rm -rf secret.xml
Secret value set
```

_ _ _

#### 第三部分: 在指定存储池中创建块设备 ####

step1:使用cinder用户在volumes池中创建10G的volume-0x02卷
```
[root@dev ~]# rbd create volume-0x02 --pool volumes --size 10240 --name client.cinder
```
_ _ _

#### 第四部分: 组装kvm虚拟机可用的块设备配置文件 ####

step1:获取系统全部mon信息

```
[root@node120 ~]# ceph mon dump | grep 6789
dumped monmap epoch 3
0: 10.160.0.120:6789/0 mon.node120
1: 10.160.0.121:6789/0 mon.node121
2: 10.160.0.122:6789/0 mon.node122
```

step2:编辑块设备xml文件(libvirt支持)

```
[root@dev ~]# virsh secret-list | grep cinder | awk '{print $1}'
1fb2b027-08d6-4739-95d1-e4f7aca367ba
```

step3:编辑块设备xml文件

```
[root@dev ~]# cat > blk.xml << EOF
<disk type='network' device='disk'>
<driver name='qemu' type='raw'/>
<auth username='cinder'>
<secret type='ceph' uuid='1fb2b027-08d6-4739-95d1-e4f7aca367ba'/>
</auth>
<source protocol='rbd' name='volumes/volume-0x02'>
<host name='10.160.0.120' port='6789'/>
<host name='10.160.0.121' port='6789'/>
<host name='10.160.0.122' port='6789'/>
</source>
<target dev='vdb' bus='virtio'/>
</disk>
EOF
```

注1: 确保auth.user与ceph用户client.{user}中的user一致且cinder secret uuid正确

注2: 确保type与块设备实际类型符合(raw)

注3: 确保name选项正确 {pool}/{volume}

注4: 确保mons信息正确

注5: 确保target dev可用

_ _ _

#### 第五部分: 以xml形式添加块设备到kvm虚拟机 ####

添加块设备并启动虚拟机

```
[root@dev ~]# virsh attach-device centos --file blk.xml --config --live
Device attached successfully

[root@dev ~]# virsh domblklist centos | grep vdb
vdb        volumes/volume-0x02
```
_ _ _

#### 第六部分: 测试磁盘读写 ####

观察虚拟机读写磁盘时ceph集群的状态

step1:虚拟机内部格式化磁盘

```
[root@localhost ~]# mkfs.xfs /dev/vdb
meta-data=/dev/vdb               isize=256    agcount=4, agsize=655360 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=0        finobt=0
data     =                       bsize=4096   blocks=2621440, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@localhost ~]# mkdir -p /data && mount -t xfs /dev/vdb /data
[root@localhost ~]# dd if=/dev/zero of=/data/test bs=1M count=1024
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 67.6022 s, 15.9 MB/s
```

step2:监控ceph集群状态

```
[root@node121 ~]# ceph -w
    cluster 914785b1-6024-4bfe-9b12-137af0ec95dc
     health HEALTH_OK
     monmap e3: 3 mons at {node120=10.160.0.120:6789/0,node121=10.160.0.121:6789/0,
                           node122=10.160.0.122:6789/0}
            election epoch 16, quorum 0,1,2 node120,node121,node122
     osdmap e156: 9 osds: 9 up, 9 in
      pgmap v31235: 768 pgs, 3 pools, 2022 MB data, 567 objects
            6984 MB used, 427 GB / 434 GB avail
                 768 active+clean
  client io 21683 B/s rd, 0 op/s

...
2017-01-04 04:57:03.517659 mon.0 [INF] pgmap v31238: 768 pgs: 768 active+clean; 2022 MB data, \
6985 MB used, 427 GB / 434 GB avail; 4057 B/s wr, 1 op/s
2017-01-04 04:57:12.023593 mon.0 [INF] pgmap v31239: 768 pgs: 768 active+clean; 2022 MB data, \
6985 MB used, 427 GB / 434 GB avail; 4760 B/s rd, 432 B/s wr, 1 op/s
2017-01-04 04:57:13.130941 mon.0 [INF] pgmap v31240: 768 pgs: 768 active+clean; 2030 MB data, \
7008 MB used, 427 GB / 434 GB avail; 4733 B/s rd, 860 kB/s wr, 1 op/s
2017-01-04 04:57:15.163476 mon.0 [INF] pgmap v31241: 768 pgs: 768 active+clean; 2034 MB data, \
7016 MB used, 427 GB / 434 GB avail; 3845 kB/s wr, 1 op/s
2017-01-04 04:57:16.177039 mon.0 [INF] pgmap v31242: 768 pgs: 768 active+clean; 2050 MB data, \
7088 MB used, 427 GB / 434 GB avail; 6509 kB/s wr, 3 op/s
```
