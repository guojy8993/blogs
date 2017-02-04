### 文档章节目录 ###
(1) Convoy插件的介绍

(2) Convoy插件的安装与配置

(3) Convoy插件的使用

(4) Convoy插件剖析

(5) Docker卷插件的API接口

(6) Docker卷插件的插件发现机制

(7) 如何自定义卷插件

(8) 附录

#### Convoy插件的介绍 ####

#### Convoy插件的安装与配置 ####

1.下载 Convoy 卷插件

```
参考附录 链接 下载卷插件
```

2.安装卷插件

```
[root@docker-comp126 opt]# ll /opt/ | grep convoy
-rw-r--r--. 1 root root 10127278 Feb  3 14:30 convoy.tar.gz

[root@docker-comp126 opt]# tar xf convoy.tar.gz
[root@docker-comp126 opt]# cp convoy/convoy convoy/convoy-pdata_tools /usr/local/bin/

[root@docker-comp126 opt]# mkdir -p /etc/docker/plugins
[root@docker-comp126 opt]# echo "unix:///var/run/convoy/convoy.sock" > /etc/docker/plugins/convoy.spec
[root@docker-comp126 opt]# cat /etc/docker/plugins/convoy.spec
unix:///var/run/convoy/convoy.sock
```

3.启动convoy服务

```
[root@docker-comp126 ~]# mkdir -p /data/docker
[root@docker-comp126 ~]# ll /data/docker
total 0
[root@docker-comp126 ~]# rm -rf /var/run/convoy/convoy.sock
[root@docker-comp126 ~]# nohup convoy daemon --drivers vfs --log --driver-opts vfs.path=/data/docker &
[1] 9462
[root@docker-comp126 ~]# nohup: ignoring input and appending output to ‘nohup.out’
```

#### Convoy插件的使用 ####

查看可用镜像并使用convoy卷插件创建卷以及启动容器

```
[root@docker-comp126 ~]# docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
10.160.0.153:5000/mysql     latest              9c1bfc3c80a1        3 months ago        575.3 MB
10.160.0.153:5000/centos7   latest              943f92dae99b        4 months ago        245 MB
centos7                     latest              943f92dae99b        4 months ago        245 MB
```

```
[root@docker-comp126 ~]# docker run -it -v volume-37a7af17df68:/test --volume-driver=convoy centos7 /bin/bash
[root@ec332a7ebede /]#
```

写入数据提示"Permission denied",这是selinux安全机制在起作用.解决办法:临时关闭selinux

```
[root@ec332a7ebede /]# echo "Data volume created by convoy" > /test/readme
bash: /test/readme: Permission denied
```

```
[root@docker-comp126 ~]# setenforce 0
[root@ec332a7ebede /]# echo "Data volume created by convoy" > /test/readme
[root@ec332a7ebede /]#
```

在宿主上，查看 vfs.path 下卷数据

```
[root@docker-comp126 ~]# cat /data/docker/volume-37a7af17df68/readme 
Data volume created by convoy
```

另外, Convoy也可单独应用实现卷/快照/备份的管理，参考附录链接

#### Convoy插件剖析 ####
![docker卷插件图示](https://github.com/guojy8993/ImageCache/blob/master/docker-volumne-plugin.jpg)
```
    Docker社区定义了一套标准的卷插件REST API, Docker自身实现了这套API的客户端,它有步骤地发现激活插件. 当Docker创建/挂载/卸
载/删除数据卷时,API客户端会向插件发送对应的REST API,由卷插件自身来真正完成创建数据卷的工作,这就是卷插件的基本原理.
    Convoy卷插件实现了一个daemon(如上图),该daemon的plugin handler模块会监听一个端口或者本地sock,用来与Docker通信,并响应
Docker卷插件REST API. 插件在接到API请求后,会通过 volume manager 模块将一个远程或本地的存储挂载到服务器端,然后将挂载到的路
径发送给Docker,然后由Docker把卷插件返回的路径挂载到容器中.可以想象,当我们使用ceph等后端存储,那么创建一个卷时,只要把远端的数
据卷挂载到本地容器中.当然,容器产生的数据也会存放在远端,这样容器的迁移共享就很容易实现了.
```

#### Docker卷插件的API接口 ####

#### Docker卷插件的插件发现机制 ####

#### 如何自定义卷插件 ####

#### 附录 ####

[Convoy插件下载](https://github.com/rancher/convoy/releases/download/v0.2.1/convoy.tar.gz)

[Convoy单独应用实现卷/快照/备份的管理](https://github.com/rancher/convoy)

[Convoy官方文档](https://github.com/rancher/convoy)

[Docker自定义卷插件](https://github.com/docker/docker/blob/master/docs/extend/plugin_api.md)




