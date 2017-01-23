### 本文档说明 ###

(1) 准备工作
i. 下载windows2k12镜像, VirtIO驱动包, CloudBase-init安装包(参考附录链接,此处略)
```
[root@cs112-04 config-2]# ll /opt/ | egrep "msi|iso|virtio"
-rw-r--r--    1 root root    35057664 Jan 22 15:24 CloudbaseInitSetup_0_9_9_x64.msi
-rw-r--r--    1 root root   155856896 Jan 22 14:16 virtio.iso
-rw-r-----    1 root root  5545527296 Jan 22 12:28 win2012.iso
```

ii.将CloudBase-init制作成ISO,稍后使用
```
[root@cs112-04 config-2]# mkisofs -R -V config-2 -o /opt/cloudinit.iso /opt/CloudbaseInitSetup_0_9_9_x64.msi
```

(2) 启动虚拟机

i. 确保服务器支持KVM虚拟化(环境配置略)

ii.新建两个linux bridge,保证后续制作的镜像的外内双网卡

iii.使用virt-install启动虚拟机


(3)驱动的加载与安装

(4)防火墙的配置

(5)CloudBase的安装与配置

(6)镜像可用性的测试

(7)附录

#### 准备工作 ####
#### 启动虚拟机 ####
#### 驱动的加载与安装 ####
#### 防火墙的配置 ####
#### CloudBase的安装与配置 ####
#### 镜像可用性的测试 ####

#### 附录 ####

[1.OpenStack官方镜像制作](http://docs.openstack.org/image-guide/windows-image.html)

[2.UserData支持powershell](https://github.com/openstack/cloudbase-init/blob/master/doc/source/userdata.rst)

[3.CloudBasae-init提供的插件与服务](https://github.com/openstack/cloudbase-init/blob/master/doc/source/plugins.rst)

[4.Windows2k12设置默认允许Ping](http://ms.n.blog.163.com/blog/static/18595352014112384724884/)

[5.CloudBase官方下载](http://www.cloudbase.it/downloads/CloudbaseInitSetup_Stable_x64.msi)

[6.VirtIO驱动官方下载](https://fedoraproject.org/wiki/Windows_Virtio_Drivers#Direct_download)

[7. 如何使用 mkisofs 制作镜像](https://github.com/guojy8993/blogs/blob/master/linux%E4%B8%8B%E7%9A%84ISO%E7%94%9F%E6%88%90%E5%B7%A5%E5%85%B7%E4%B8%8E%E4%BD%BF%E7%94%A8.md)
