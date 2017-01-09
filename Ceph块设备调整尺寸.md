### 本文档内容 ###

1.针对已挂载使用的ceph块设备调整尺寸

2.客户机内部在线调更新块设备大小
_ _ _

#### 第一部分 针对已挂载使用的ceph块设备调整尺寸 ####
```
[root@dev ~]# rbd showmapped | grep rbd
0  volumes base  -    /dev/rbd0

[root@dev ~]# df | grep rbd
/dev/rbd0 10475520 33344 10442176 1% /mnt/rbd
```

调整尺寸前

```
[root@dev ~]# rbd info volumes/base --id cinder
rbd image 'base':
	size 10240 MB in 2560 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.ae96238e1f29
	format: 2
	features: layering
	flags:
```

调整尺寸

```
[root@dev ~]# rbd resize --pool volumes --image base --size 5120 --name client.cinder
rbd: shrinking an image is only allowed with the --allow-shrink flag

[root@dev ~]# rbd resize --pool volumes --image base --size 5120 --name client.cinder --allow-shrink
Resizing image: 100% complete...done.
```

注1: size单位MB

调整尺寸后
```
[root@dev ~]# rbd info volumes/base --name client.cinder
rbd image 'base':
	size 5120 MB in 1280 objects
	order 22 (4096 kB objects)
	block_name_prefix: rb.0.aeab.238e1f29
	format: 1
```
_ _ _

#### 第二部分 客户机内部在线调更新块设备大小 ####
```
[root@dev ~]# dmesg | grep -i capacity
[root@dev ~]# xfs_growfs -d /mnt/rbd
meta-data=/dev/rbd0              isize=256    agcount=17, agsize=162816 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=0        finobt=0
data     =                       bsize=4096   blocks=2621440, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data size 1310720 too small, old size is 2621440
```

注1: 原/dev/rbd0被格式化为xfs文件系统 - XFS文件系统支持在线调整大小
