#### 第一部分:LVM相关特殊"名词" ####

```
PV 物理卷。是逻辑卷(LVM)最底层概念,是LVM逻辑存储块,与磁盘分区是逻辑对应关系.
   LVM管理通过pvcreate将物理磁盘分区转化为物理卷,通过组合物理卷成为卷组.
VG 卷组。是LVM逻辑上的磁盘设备。个人见解,是对碎片化的物理设备的整合与资源池化。
PE 物理长度。将物理卷组合为卷组后划分的最小存储单元。默认4M。
LV 逻辑卷。是LVM逻辑意义上的分区。可以从指定卷组(甚至指定物理卷提取容量创建逻辑卷)，
   最后对其格式化并挂载使用。本质上是对池化资源的再划分。
```

#### 第二部分:硬件块设备/PV/VG/LV之间的关联 ####

```
            转化
硬件块设备00--->PV(物理卷)---+
            转化            | 整合为资源池           分割资源
硬件块设备01--->PV(物理卷)---+----------->VG(卷组)--------------> LV0,LV1,LV2,....
...                         |
```

#### 第三部分:PV/VG/LV管理 ####
(1) 物理卷管理

```
[root@frontend ~]# parted /dev/sdb print
Model: VMware Virtual disk (scsi)
Disk /dev/sdb: 53.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:
Number  Start   End     Size    File system  Name     Flags
 1      1049kB  10.0GB  9999MB  xfs          primary
 2      10.0GB  20.0GB  9999MB  xfs          primary
 3      20.0GB  30.0GB  10.0GB               primary
 4      30.0GB  40.0GB  10.0GB               primary
 5      40.0GB  50.0GB  10.0GB               primary  
[root@frontend ~]# pvcreate /dev/sdd1
Physical volume "/dev/sdd1" successfully created
[root@frontend ~]# pvcreate /dev/sdd{2..5}
Physical volume "/dev/sdd2" successfully created
Physical volume "/dev/sdd3" successfully created
Physical volume "/dev/sdd4" successfully created
Physical volume "/dev/sdd5" successfully created

[root@frontend ~]# pvs
  PV         VG     Fmt  Attr PSize  PFree 
  /dev/sda2  centos lvm2 a--  49.51g 44.00m
  /dev/sdb1  sddvg  lvm2 a--   9.31g  6.53g
  /dev/sdb2  sddvg  lvm2 a--   9.31g  9.31g
  /dev/sdb3  sddvg  lvm2 a--   9.31g  9.31g
  /dev/sdb4  sddvg2 lvm2 a--   9.31g  9.31g
  /dev/sdb5  sddvg2 lvm2 a--   9.31g  7.31g
  /dev/sdd1         lvm2 ---   9.31g  9.31g
  /dev/sdd2         lvm2 ---   9.31g  9.31g
  /dev/sdd3         lvm2 ---   9.31g  9.31g
  /dev/sdd4         lvm2 ---   9.31g  9.31g
  /dev/sdd5         lvm2 ---   9.31g  9.31g
```

(2) 卷组管理

```
[root@frontend ~]# vgcreate sddvg /dev/sdb1 /dev/sdb2 /dev/sdb3
  Volume group "sddvg" successfully created
# 指定PE创建卷组
[root@frontend ~]# vgcreate sddvg2 -s 16M /dev/sdb4 /dev/sdb5
  Volume group "sddvg2" successfully created
[root@frontend ~]# vgs | grep sdd
  sddvg    3   2   0 wz--n- 27.93g 25.15g
  sddvg2   2   1   0 wz--n- 18.62g 16.62g
```

(3) 逻辑卷管理

```
[root@frontend ~]# lvcreate -L 2G -n sddvg_lv001 sddvg
  Logical volume "sddvg_lv001" created.
# 使用指定卷组的指定物理卷创建逻辑卷(可以实现类似ceph SSD pool的东西)
[root@frontend ~]# lvcreate -L 2G -n sddvg_lv002 sddvg2 /dev/sdb5
  Logical volume "sddvg_lv002" created. 
[root@frontend ~]# lvcreate -l 200 -n sddvg_lv003 sddvg
  Logical volume "sddvg_lv003" created.
[root@frontend ~]# lvdisplay | grep "LV Name" | grep sdd
  LV Name                sddvg_lv002
  LV Name                sddvg_lv001
  LV Name                sddvg_lv003

# 逻辑卷扩容
[root@frontend ~]# lvresize /dev/sddvg/sddvg_lv001 -L +4G
  Size of logical volume sddvg/sddvg_lv001 changed from 2.00 GiB (1024 extents) to 6.00 GiB (1536 extents).
  Logical volume sddvg_lv001 successfully resized
```
#### 第四部分:扩展 ###
[LVM有thin/fat之别](http://www.searchvirtual.com.cn/showcontent_87194.htm)

各种高级应用参考openstack子项目cinder的cinder/brick/local_dev/lvm.py
