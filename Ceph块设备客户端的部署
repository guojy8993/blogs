### 本文档内容提要 ###
1.在目标机器安装ceph及其依赖程序

2.为客户端创建对ceph集群用户,并设置其读写权限以及可操作存储池

3.将ceph用户的密钥文件保存到客户端机器

4.测试指定ceph用户的可用性以及权限

_ _ _

#### 第一部分: 在目标机器安装ceph及其依赖程序 ####

说明:有多种途径来达成

(1) 在ceph集群的主控节点上使用ceph-deploy install推送安装

(2) 使用预先下载的rpm包安装

(3) yum安装

方法1: ceph-deploy推送安装

step1: 设置ceph-deploy节点(node120)到客户端机器(node144)的免验证登录
```
[root@node120 ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub root@10.160.0.144
```
step2: 使用ceph-deploy向客户端机器推送安装
```
[root@node120 ~]# ceph-deploy --username root install 10.160.0.144
```
方法2: rpm安装包安装

step1: 去官方源下载对应版本的rpm包

```
boost-program-options-1.53.0-25.el7.x86_64.rpm
hdparm-9.43-5.el7.x86_64.rpm
leveldb-1.7.0-2.el6.x86_64.rpm
libcephfs1-0.94.9-0.el7.x86_64.rpm
librados2-0.94.9-0.el7.x86_64.rpm
librbd1-0.94.9-0.el7.x86_64.rpm
libtcmalloc4-2.2-3.3.1.x86_64.rpm
libunwind-1.1-5.el7.x86_64.rpm
python-cephfs-0.94.9-0.el7.x86_64.rpm
python-rados-0.94.9-0.el7.x86_64.rpm
python-rbd-0.94.9-0.el7.x86_64.rpm
redhat-lsb-core-4.1-27.el7.centos.1.x86_64.rpm
ceph-common-0.94.9-0.el7.x86_64.rpm
ceph-0.94.9-0.el7.x86_64.rpm
ceph-deploy-1.5.36-0.noarch.rpm
ceph-radosgw-0.94.9-0.el7.x86_64.rpm
```

step2: 使用rpm命令安装软件,并按照错误提示安装其依赖
_ _ _

#### 第二部分: 为客户端创建对ceph集群用户,并设置其读写权限以及可操作存储池 ####

示例: 创建rbd用户,允许rwx存储池images
```
[root@node120 ~]# ceph auth get-or-create client.cinder mon 'allow r' \
osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, \
allow rwx pool=vms, allow rwx pool=images'
[client.cinder]
	key = AQCbbWxYPM5tBBAARTHNRPzuR5A2t/peKuFmIA==
[root@node120 ~]# ceph auth get-or-create client.cinder 
[client.cinder]
	key = AQCbbWxYPM5tBBAARTHNRPzuR5A2t/peKuFmIA==
```
_ _ _

#### 第三部分: 将ceph用户的密钥文件保存到客户端机器 ####
```
[root@node120 ~]# ceph auth get-or-create client.cinder | ssh root@10.160.0.144 \
tee /etc/ceph/ceph.client.cinder.keyring
[client.cinder]
	key = AQCbbWxYPM5tBBAARTHNRPzuR5A2t/peKuFmIA==
```
_ _ _

#### 第四部分: 测试指定ceph用户的可用性以及权限 ####

示例: 使用cinder用户在ceph集群的volumes池中创建大小为10240MB的块设备volume-0x01
```
[root@dev ~]# rbd create volume-0x01 --pool volumes --name client.cinder --size 10240
[root@dev ~]# rbd ls --pool volumes --name client.cinder
volume-0x01
[root@dev ~]# rbd info volume-0x01 --pool volumes --name client.cinder
rbd image 'volume-0x01':
	size 10240 MB in 2560 objects
	order 22 (4096 kB objects)
	block_name_prefix: rb.0.ad4c.238e1f29
	format: 1
```
