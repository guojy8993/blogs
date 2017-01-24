### 快照管理 ###
(1) 创建快照(virsh snapshot-create-as),帮助信息如下
```
[root@agent144 KY009]# virsh help snapshot-create-as
  NAME
    snapshot-create-as - Create a snapshot from a set of args
  SYNOPSIS
    snapshot-create-as <domain> [--name <string>] [--description <string>] [--print-xml]
    [--no-metadata] [--halt] [--disk-only] [--reuse-external] [--quiesce] [--atomic] [--live]
    [--memspec <string>] [[--diskspec] <string>]...
  DESCRIPTION
    Create a snapshot (disk and RAM) from arguments
```
> **NOTE:**
> 虚拟机快照创建选项说明:

> 必选项:虚拟机名字                [--domain] <string>  domain name, id or uuid

> 可选项:快照名字                  --name <string>  name of snapshot

> 可选项:快照描述信息              --description <string>  description of snapshot

> 可选项:只打印xml不创建           --print-xml  print XML document rather than create

> 可选项:不创建元数据              --no-metadata take snapshot but create no metadata

> 可选项:快照创建后暂停机器        --halt   halt domain after snapshot is created

> 可选项:仅创建磁盘快照,不创建机器快照    --disk-only  capture disk state but not vm state

> 可选项:再利用外部文件(似乎是 external snapshot)  --reuse-external  reuse any existing external files

> 可选项:客户机文件系统静默              --quiesce   quiesce guest's file systems

> 可选项:原子操作       --atomic   require atomic operation

> 可选项:热操作:        --live     take a live snapshot

> 可选项:内存选项       --memspec <string>  memory attributes: [file=]name[,snapshot=type]

> 可选项:磁盘选项       [--diskspec] <string>  disk attributes: disk[,snapshot=type][,driver=type][,file=name]

## 下面是具体使用场景展示 ##
# !!! agent144 是宿主,gainetKXs 是客户机 !!!

# step1: 查看 KY-vxlan100-A 虚拟机信息
# 只有系统盘
[root@agent144 KY-vxlan100-A]# virsh domblklist  KY-vxlan100-A
Target     Source
------------------------------------------------
vda        /data/instance/KY-vxlan100-A/system

# step2: 对 KY-vxlan100-A 虚拟机创建快照
# 查看虚拟机 /root/snap.txt 中内容
[root@gainetKXs ~]# touch snap.txt && echo "system disk only" >> snap.txt
[root@gainetKXs ~]# cat /root/snap.txt 
system disk only

# 创建快照
[root@agent144 KY-vxlan100-A]# virsh snapshot-create-as --domain KY-vxlan100-A \
                                  --description "system disk only and snap.txt exists in /root "
Domain snapshot 1462929799 created
[root@gainetKXs ~]# echo "system reversed" >> snap.txt 
[root@gainetKXs ~]# cat /root/snap.txt 
system disk only
system reversed

# 查看快照列表
[root@agent144 KY-vxlan100-A]# virsh snapshot-list --domain KY-vxlan100-A
 Name                 Creation Time             State
------------------------------------------------------------
 1462929799           2016-05-10 21:23:19 -0400 running

# step3: 在 KY-vxlan100-A 内操作文件
# 查看 /root/snap.txt文件内容
# step4: 执行快照还原,并查看 /root/snap.txt 中内容
[root@agent144 KY-vxlan100-A]# virsh snapshot-revert --domain KY-vxlan100-A --snapshotname 1462929799
[root@gainetKXs ~]# ls
snap.txt
[root@gainetKXs ~]# cat snap.txt 
system disk only

# step5: 为磁盘挂载新的数据盘
[root@agent144 KY-vxlan100-A]# qemu-img create -f qcow2 data 1G
Formatting 'data', fmt=qcow2 size=1073741824 encryption=off cluster_size=65536 \
lazy_refcounts=off refcount_bits=16
[root@agent144 KY-vxlan100-A]# virsh attach-disk --domain KY-vxlan100-A \
                                  --source /data/instance/KY-vxlan100-A/data  \
                                  --target vdb --targetbus virtio --driver qemu  \
                                  --subdriver qcow2 --cache writeback --live --config \
                                  --sourcetype file
Disk attached successfully
[root@agent144 KY-vxlan100-A]# virsh domblklist KY-vxlan100-A
Target     Source
------------------------------------------------
vda        /data/instance/KY-vxlan100-A/system
vdb        /data/instance/KY-vxlan100-A/data

# step6: 在虚拟机内格式化新加磁盘,挂载,并创建文件
[root@gainetKXs ~]# fdisk -l | grep Disk
Disk /dev/vda: 10.7 GB, 10737418240 bytes, 20971520 sectors
Disk label type: dos
Disk identifier: 0x000de2c8
Disk /dev/mapper/centos-root: 9093 MB, 9093251072 bytes, 17760256 sectors
Disk /dev/mapper/centos-swap: 1073 MB, 1073741824 bytes, 2097152 sectors
Disk /dev/vdb: 1073 MB, 1073741824 bytes, 2097152 sectors

[root@gainetKXs ~]# mkfs.xfs -f /dev/vdb
meta-data=/dev/vdb               isize=256    agcount=4, agsize=65536 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=0        finobt=0
data     =                       bsize=4096   blocks=262144, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@gainetKXs ~]# mkdir -p /home/snap
[root@gainetKXs ~]# mount -t xfs /dev/vdb /home/snap
[root@gainetKXs ~]# echo "data disk & /home/snap/snap.txt" > /home/snap/snap.txt
[root@gainetKXs ~]# cat /home/snap/snap.txt
data disk & /home/snap/snap.txt

# step7: 对虚拟机创建快照
# 两块磁盘,且将数据盘挂载到 /home/snap/,并创建文件测试磁盘可用
[root@agent144 KY-vxlan100-A]# virsh snapshot-create-as --domain KY-vxlan100-A --description \
                                  "home snapshot and data disk"
Domain snapshot 1462931376 created
# step8: 列出可用快照
[root@agent144 KY-vxlan100-A]# virsh snapshot-list  --domain KY-vxlan100-A
 Name                 Creation Time             State
------------------------------------------------------------
 1462931042           2016-05-10 21:44:02 -0400 running
 1462931376           2016-05-10 21:49:36 -0400 running
 
# step9: 将新加数据盘卸载,并执行快照还原
[root@agent144 KY-vxlan100-A]# virsh detach-disk --domain KY-vxlan100-A --target vdb --live --config
Disk detached successfully
[root@gainetKXs ~]# fdisk -l | grep Disk
Disk /dev/vda: 10.7 GB, 10737418240 bytes, 20971520 sectors
Disk label type: dos
Disk identifier: 0x000de2c8
Disk /dev/mapper/centos-root: 9093 MB, 9093251072 bytes, 17760256 sectors
Disk /dev/mapper/centos-swap: 1073 MB, 1073741824 bytes, 2097152 sectors

[root@agent144 KY-vxlan100-A]# virsh snapshot-revert --domain KY-vxlan100-A --snapshotname 1462931376
error: revert requires force: Target domain disk count 2 does not match source 1
[root@agent144 KY-vxlan100-A]# virsh snapshot-revert --domain KY-vxlan100-A --snapshotname 1462931376 --force

# step10: 查看此时虚拟机磁盘列表,以及挂载文件夹内的文件
# 磁盘自动重新挂载
# 磁盘内文件还在
[root@agent144 KY-vxlan100-A]# virsh domblklist KY-vxlan100-A
Target     Source
------------------------------------------------
vda        /data/instance/KY-vxlan100-A/system
vdb        /data/instance/KY-vxlan100-A/data
[root@gainetKXs ~]# fdisk -l | grep Disk
Disk /dev/vda: 10.7 GB, 10737418240 bytes, 20971520 sectors
Disk label type: dos
Disk identifier: 0x000de2c8
Disk /dev/mapper/centos-root: 9093 MB, 9093251072 bytes, 17760256 sectors
Disk /dev/mapper/centos-swap: 1073 MB, 1073741824 bytes, 2097152 sectors
Disk /dev/vdb: 1073 MB, 1073741824 bytes, 2097152 sectors
[root@gainetKXs ~]# cat /home/snap
snap/     snap.txt  
[root@gainetKXs ~]# cat /home/snap/snap.txt 
data disk & /home/snap/snap.txt

# step11: 将数据盘文件修改名字,重新执行快照还原
# step12: 查看虚拟机快照信息以及磁盘的快照信息

# 整机快照的信息
[root@agent144 KY-vxlan100-A]# virsh snapshot-info --domain KY-vxlan100-A --snapshotname 1462931376
Name:           1462931376
Domain:         KY-vxlan100-A
Current:        yes
State:          running
Location:       internal
Parent:         1462931042
Children:       0
Descendants:    0
Metadata:       yes

# 数据盘快照的信息
[root@agent144 KY-vxlan100-A]# qemu-img info /data/instance/KY-vxlan100-A/data
image: /data/instance/KY-vxlan100-A/data
file format: qcow2
virtual size: 1.0G (1073741824 bytes)
disk size: 24M
cluster_size: 65536
Snapshot list:
ID        TAG                 VM SIZE                DATE       VM CLOCK
2         1462873758                0 2016-05-10 05:49:18   02:12:36.291
3         1462931042                0 2016-05-10 21:44:02   00:43:34.510
4         1462931376                0 2016-05-10 21:49:36   00:49:07.673
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false

# 系统盘快照的信息
[root@agent144 KY-vxlan100-A]# qemu-img info /data/instance/KY-vxlan100-A/system
image: /data/instance/KY-vxlan100-A/system
file format: qcow2
virtual size: 10G (10737418240 bytes)
disk size: 11G
cluster_size: 65536
Snapshot list:
ID        TAG                 VM SIZE                DATE       VM CLOCK
2         1462930969             229M 2016-05-10 21:42:49   00:42:24.459
3         1462931042             229M 2016-05-10 21:44:02   00:43:34.510
4         1462931376             230M 2016-05-10 21:49:36   00:49:07.673
Format specific information:
    compat: 1.1
    lazy refcounts: true
    refcount bits: 16
    corrupt: false

注意事项:
# raw 格式磁盘是不支持快照创建的
[root@agent144 KY009]# virsh snapshot-create-as --domain KY009 --description "Snapshot cirros "
error: unsupported configuration: internal snapshot for disk vda unsupported for storage type raw
# 有快照的磁盘是不允许扩容的
[root@agent144 KY-vxlan100-A]# qemu-img resize -q /data/instance/KY-vxlan100-A/data +1G
qemu-img: Can't resize an image which has snapshots
qemu-img: This image does not support resize
# 有快照的而使用非共享存储的机器没法迁移
[root@agent144 KY-vxlan100-A]# sh migrate-with-local-volume 
error: Requested operation is not valid: cannot migrate domain with 1 snapshots


快照创建与还原,参考文档:
1.http://www.it165.net/os/html/201309/6189.html             # 快照-实践篇
2.http://blog.chinaunix.net/uid-20794164-id-3989713.html    # 快照-理论篇
3.http://blog.chinaunix.net/uid-20940095-id-4752643.html    # openstack中的虚拟机快照功能
