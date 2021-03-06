### 文档内容提要 ###
1.使用指定用户上传本地模板到ceph并指定镜像"允许克隆与分层"的特性

2.对指定模板创建快照并设置保护

3.从镜像克隆出新的系统模板供虚拟机使用

4.修改虚拟机配置文件,从指定的克隆模板启动

5.将克隆的模板依赖的镜像与快照合并,解除对父镜像与快照的依赖

6.取消快照保护并删除快照

_ _ _

#### 第一部分 使用指定用户上传本地模板到ceph并指定镜像"允许克隆与分层"的特性 ####
```
[root@dev ~]# rbd import /data/image/centos7.qcow2 CentOS7-Layer-Support --pool images \
                  --size 10240 --image-format 2 --image-features +1 --name client.cinder
[root@dev ~]# rbd info images/CentOS7-Layer-Support 
rbd image 'CentOS7-Layer-Support':
	size 1988 MB in 497 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.add32ae8944a
	format: 2
	features: layering
	flags:
```

注1:创建/导入镜像允许特性"克隆与分层":image-format值为2

_ _ _

#### 第二部分 对指定模板创建快照并设置保护 ####
```
[root@dev ~]# rbd snap create images/CentOS7-Layer-Support@Snap_CentOS7 --name client.cinder
[root@dev ~]# rbd snap protect images/CentOS7-Layer-Support@Snap_CentOS7 --name client.cinder
```

_ _ _

#### 第三部分 从镜像克隆出新的系统模板供虚拟机使用 ####
```
[root@dev ~]# rbd clone images/CentOS7-Layer-Support@Snap_CentOS7 volumes/centos-bootable \
--name client.cinder
[root@dev ~]# rbd info volumes/centos-bootable
rbd image 'centos-bootable':
	size 1988 MB in 497 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.ae3c2eb141f2
	format: 2
	features: layering
	flags: 
	parent: images/CentOS7-Layer-Support@Snap_CentOS7
	overlap: 1988 MB
```
_ _ _

#### 第四部分 修改虚拟机配置文件,从指定的克隆模板启动 ####
```
[root@dev ~]# virsh edit centos
...
<disk type='network' device='disk'>
	<driver name='qemu' type='qcow2'/>
	<auth username='cinder'>
		<secret type='ceph' uuid='1fb2b027-08d6-4739-95d1-e4f7aca367ba'/>
	</auth>
	<source protocol='rbd' name='volumes/centos-bootable'>
		<host name='10.160.0.120' port='6789'/>
		<host name='10.160.0.121' port='6789'/>
		<host name='10.160.0.122' port='6789'/>
	</source>
	<backingStore/>
	<target dev='vda' bus='virtio'/>
	<alias name='virtio-disk0'/>
	<address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
</disk>
...
```
_ _ _

#### 第五部分 将克隆的模板依赖的镜像与快照合并,解除对父镜像与快照的依赖 ####
```
[root@dev ~]# rbd flatten volumes/centos-bootable --name client.cinder
Image flatten: 100% complete...done.
```

注1:这是个耗时操作

```
[root@dev ~]# rbd info volumes/centos-bootable --name client.cinder
rbd image 'centos-bootable':
	size 1988 MB in 497 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.ae3c2eb141f2
	format: 2
	features: layering
	flags:
```

注1: 无parent,已经解除依赖

_ _ _

#### 第六部分 取消快照保护并删除快照 ####
```
[root@dev ~]# rbd snap unprotect images/CentOS7-Layer-Support@Snap_CentOS7 --name client.cinder
[root@dev ~]# rbd snap rm images/CentOS7-Layer-Support@Snap_CentOS7 --name client.cinder
[root@dev ~]# rbd snap ls --pool images --image CentOS7-Layer-Support --name client.cinder
[root@dev ~]#
```
