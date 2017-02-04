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

2.安装Convoy卷插件

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
    卷插件的接口一共有6个,这些接口都是REST API,是由Docker作为Client发送给卷插件的每个接口都有不同的含义.
 ```
(1) Plugin.Activate  该接口用于激活一个插件,是Docker和卷插件的握手报文.
```
```
(2) VolumeDriver.Create  该接口用于创建一个卷,Docker会发送卷名称和参数给卷插件,卷插件会根据Docker发送过来的参数创建一个卷,
并和这个卷名称关联
```
```
(3) VolumeDriver.Mount  该接口用于挂载一个卷到本机,Docker会把卷名称或卷参数发送给插件,插件返回一个本地路径给Docker,这个路
径就是卷所在的位置.Docker在创建容器的时候,会将这个(本地)路径挂载到容器中.
```
```
(4) VolumeDriver.Path  一个卷创建成功后,Docker会调用Path API来获取这个卷的路径,随后Docker通过调用Mount API,将这个卷挂载到
本机.
```
```
(5) VolumeDriver.Umount  当容器退出时,Docker daemon会发送Umount API给插件,通知插件这个卷不再被使用,插件可以对该卷做些清理
工作(例如,引用计数减一,不同的插件行为不同)
```
```
(6) VolumeDriver.Remove  删除特定的卷时调用,例如当运行"docker rm -v"命令时,Docker会将该API发送给请求插件.
```
```
注意:  整个卷插件系统是通过卷名称(Volume Name)来管理的.比如创建卷API时,Docker会发送一个卷名称给插件,插件返回成功,同理Docker
也会发送VolumePath到插件(参数也是卷名称),插件则将该卷的挂载路径发送给Docker.而且插件的CLI也是通过卷名称来备份/管理/创建卷的.
```

#### Docker卷插件的插件发现机制 ####
```
    那么行文到此,有个疑问尚待解决: Docker作为客户端是如何知道插件的监听地址呢? 在该部分就"Docker的数据卷插件发现机制"进行分
析.
```
```
    首先,Docker是通过JSON配置文件,Unix Domain Socket(域套接字或Spec后缀的配置文件来查询插件的地址). 在"Convoy插件的安装与
配置"以及"Convoy插件的使用"部分,运行"docker run"命令,在该命令中启动容器时可以加 --volume-driver=convoy 参数,这个 volume-
driver后面加的就是插件的名称. 当docker发现该参数的设置时,就会去几个文件夹下搜索Unix套接字(convoy.sock)或者相应的配置文件(
convoy.spec或convoy.json)
```
```
    Docker首先(!!此处特别注意!!)会搜索 /run/docker/plugins 路径下的套接字文件,这个文件名必须与Volume Driver名称一致(例如,
volume driver为convoy时,那么就搜索 convoy.sock).如果找不到,就会依次到 /etc/docker/plugins 和 /usr/lib/docker/plugins 里
面继续搜索.如果搜索到就加载该插件. 如果搜索不到, 那么docker run命令将返回错误.
```
```
    注意: 套接字只能存放在 /run/docker/plugins 目录下.另外,如果卷插件不使用域套接字,而是使用spec或json配置文件,那么文件里
面就可以配置插件的URL,如果是https链接,还可以指定https的证书路径.
```
```
    注意: docker daemon与卷插件convoy daemon之间基于域套接字的通信是很典型的"本地主机跨进程通信".
```

#### 如何自定义卷插件 ####
```
    查看文档"Docker自定义卷插件"(附录链接)一文的 "Create a VolumeDriver",根据接口约定的request path info(诸如 /Volume
Driver.Create) 以及 request body 和 response body,我们可以自己实现一个 plugin handler(服务端程序).
```
```
    具体是这样的,plugin handler程序解析 request 获取 path info,以此将请求dispatch到某具体接口的实现(VolumeDriver.Cre
ate).对于某接口的具体实现,则是解析request body,获取POST参数(目前卷插件约定的6个接口全部是POST请求),解析之，然后根据自定义
卷插件的插件实现完成对应的API调用，并按约定返回指定的 response body.
```
```
    一旦该server程序实现全部功能,即可按照前文"Docker卷插件的插件发现机制"以及"安装Convoy卷插件"一样配置插件,然后测试对应功
能是否可用.
```
```
    基于以上说明,我使用python实现服务程序以及使用ceph rbd作为存储后端,具体实现参考附录链接"docker使用ceph rbd存储后端".
```
#### 附录 ####

[Convoy插件下载](https://github.com/rancher/convoy/releases/download/v0.2.1/convoy.tar.gz)

[Convoy单独应用实现卷/快照/备份的管理](https://github.com/rancher/convoy)

[Convoy官方文档](https://github.com/rancher/convoy)

[窥探docker中的volume plugin内幕](http://blog.csdn.net/haleycomet/article/details/52452680)

[Docker自定义卷插件](https://docs.docker.com/engine/extend/plugins_volume/)

[Docker Plugin API Design](https://docs.docker.com/engine/extend/plugin_api/)

[Docker自定义卷插件之使用ceph rbd存储后端](https://github.com/rancher/convoy/releases/download/v0.2.1/convoy.tar.gz)
