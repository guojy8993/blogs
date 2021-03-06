### 文档内容提要 ###
1. Cloudinit对ConfigDrive数据组织与结构的要求

___
#### Cloudinit对ConfigDrive目录组织与数据结构的要求 ####
       虽然按照Cloud-Init官方文档以及OpenStack相关文档的介绍，ConfigDrive内部复杂的文件系统组织以及数据结构，相当丰富但也很复杂。结合笔者测试，归结了最简的文件结构，参考下文：
```
[root@dev config-2]# pwd
/root/config-2
[root@dev config-2]# tree
.
└── openstack
    ├── content
    │   ├── 0000
    │   ├── 0001
    │   ├── 0002
    │   └── 0003
    └── latest
        ├── meta_data.json
        ├── user_data
        └── vendor_data.json

3 directories, 7 files
```
(1)  最重要的是meta_data.json，作用:

i. 从 configdrive 中 content 目录拷贝文件为虚拟机特定文件

ii. 定义了虚拟机的基本配置信息

``` 
{
    "files": [
        {
            "path": "/root/userdata",
            "content_path": "/content/0000"
        },
        {
            "path": "/root/tokens",
            "content_path": "/content/0001"
        },
        {
            "path": "/root/openvswitch.rpm",
            "content_path": "/content/0002"
        },
        {
            "path": "/etc/sysconfig/network-scripts/ifcfg-eth0",
            "content_path": "/content/0003"
        }
    ],
    "hostname": "cloudinit-devops.novalocal",
    "launch_index": 0,
    "name": "cloudinit-devops",
    "uuid": "c5d1d5bd-4847-45b0-8ee4-8f3075900bfc"
}
```
> **NOTE：**

> hostname与name是虚拟机配置的必须信息

> uuid 是必须的，经测试，如果没有该项则注入就失败

> launch_index 建议使用0作为默认

> ** 支持通过 admin_pass 进行密码注入 **

> files列表定义了需要从 ./openstack/content/ 拷贝到虚拟机的指定文件

> files的典型应用包括：

> (1)  网卡配置，域名解析，本地解析，yum源，known_hosts等

> (2)  拷贝软件安装包(存在content下命名为"\d{4}")到虚拟机，后续使用user_data脚本安装

(2) 次重要的当属于user_data，其实就是用户自定义的可执行脚本，作用：完成用户指定的操作

以笔者测试环境为例：

```
#!/bin/bash
echo root | passwd --stdin root
echo "10.160.0.116 master116.openstack.org" >> /etc/hosts
rpm -ivh /root/openvswitch.rpm
systemctl start openvswitch
systemctl restart network
```

功能包括:

i.  修改系统密码

ii. 添加域名解析

iii. 安装并启动openvswitch软件

iv. 重启网络服务

> **NOTE:**

> 结合meta-data.json来看:user-data的执行时机是在CloudInit优先完成meta-data.json的工作(拷贝文件，设置机器名等)之后

> user-data是明文脚本，并未使用base64加密

> user-data经自测支持shell，python，但需要定义好文件头；bat以及powershell官方支持，但此处未测试

(3) 最后是content目录，存储需要拷贝到虚拟机的各色文件，但是对命名有特殊要求,符合正则 "\d{4}"

> **NOTE:**

> ConfigDrive对卷标有特殊要求，需要设定为 config-2:

> 以笔者测试环境为例: mkisofs -R -V config-2 -o /root/configdrive.iso /root/config-2/

> 后续使用virsh change-media命令插入虚拟机光驱设备(略)

> 在虚拟机内部iso被识别为: /dev/disk/by-label/config-2

> CloudInit识别约定设备作为ConfigDrive，读取其中配置实现系统配置

按照如上组织ConfigDrive，然后以之启动虚拟机：

![CloudInit拷贝文件](https://github.com/guojy8993/blogs/blob/master/cloudinit-metadata.jpg)

![CloudInit用户脚本](https://github.com/guojy8993/blogs/blob/master/cloudinit-userdata-file.jpg)

![CloudInit用户脚本执行结果](https://github.com/guojy8993/blogs/blob/master/cloudinit-userdata.jpg)
___

#### 参考资料 ####
[1. CloudInit初始化时机](http://cloudinit.readthedocs.io/en/latest/topics/boot.html)

[2. UserData格式](http://cloudinit.readthedocs.io/en/latest/topics/format.html#example)

[3. CloudInit文件布局](http://cloudinit.readthedocs.io/en/latest/topics/dir_layout.html)

[4. Cloudinit镜像制作](https://github.com/guojy8993/blogs/blob/master/OpenStack%E9%95%9C%E5%83%8F%28%E5%9F%BA%E4%BA%8ECentOS7%29%E7%9A%84%E5%88%B6%E4%BD%9C%E4%B8%8E%E8%AF%B4%E6%98%8E)

版权所有，转载注明出处。如有问题联系 guojy8993@163.com 
