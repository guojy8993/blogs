### 文档内容提要 ###
1.ceph创建块设备,并进行rbd映射到测试服务器

2.在块设备文件系统写入标记文件

3.对块设备作快照

4.删除标记文件,回滚快照

5.刷新挂载的块设备,检查标记文件

6.删除快照

_ _ _

#### 第一部分 ceph创建块设备,并进行rbd映射到测试服务器 ####

step1: 参考"Ceph块设备映射"进行操作

#### 第二部分 在块设备文件系统写入标记文件 ####
```
[root@dev rbd]# rbd showmapped
id pool    image snap device    
0  volumes base  -    /dev/rbd0

[root@dev rbd]# df | grep rbd
/dev/rbd0                10475520   33348  10442172   1% /mnt/rbd

[root@dev rbd]# pwd
/mnt/rbd

[root@dev rbd]# echo "Contents pre-snapshoting ..." >> note.txt
[root@dev rbd]# cat note.txt 
Contents pre-snapshoting ...
```
_ _ _

#### 第三部分 对块设备作快照 ####
```
[root@dev ~]# rbd snap create volumes/base@snap_base --name client.cinder
[root@dev ~]# rbd snap ls volumes/base --name client.cinder
SNAPID NAME          SIZE 
     4 snap_base 10240 MB
```

#### 第四部分 删除标记文件,回滚快照 ####
```
[root@dev rbd]# rm -rf note.txt 
[root@dev rbd]# ll
total 0
[root@dev ~]# rbd snap rollback volumes/base@snap_base --name client.cinder
Rolling back to snapshot: 100% complete...done.
```

#### 第五部分 刷新挂载的块设备,检查标记文件 ####
```
[root@dev rbd]# cd ../
[root@dev mnt]# umount rbd
[root@dev mnt]# mount -t xfs /dev/rbd0 rbd
[root@dev mnt]# cd rbd
[root@dev rbd]# ls
note.txt
[root@dev rbd]# cat note.txt 
Contents pre-snapshoting ...
```

#### 第六部分 删除快照 ####
```
[root@dev ~]# rbd snap rm volumes/base@snap_base --name client.cinder
[root@dev ~]# rbd snap ls volumes/base --name client.cinder
[root@dev ~]#
```
note: 使用rbd snap purge批量删除指定镜像的全部快照,例如,
```
rbd snap purge volumes/base --name client.cinder
```
