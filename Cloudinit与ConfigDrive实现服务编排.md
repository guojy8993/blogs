### 文档内容提要 ###
1. Cloudinit介绍
2. ConfigDrive介绍
3. Cloudinit对ConfigDrive数据组织与结构的要求
4. Cloudinit结合ConfigDrive实现服务编排

___
#### Cloudinit介绍 ####
___
#### ConfigDrive介绍 ####
___
#### Cloudinit对ConfigDrive目录组织与数据结构的要求 ####
       虽然按照Cloud-Init官方文档以及OpenStack相关文档的介绍，ConfigDrive内部复杂的文件系统组织以及数据结构，相当丰富繁杂。结合笔者测试，归结了
最简的文件结构，参考下文：
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
(1) 最重要的是meta_data.json，作用:

i. 从 configdrive 中 content 目录拷贝文件为虚拟机特定文件

ii.定义了虚拟机的基本配置信息

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
> **NOTE**
> hostname与name是虚拟机配置的必须信息
> uuid 是必须的，经测试，如果没有该项则注入就失败
> launch_index 建议使用0作为默认
> files列表定义了需要从 ./openstack/content/ 拷贝到虚拟机的指定文件
> files的典型应用包括：
> (1) 网卡配置，域名解析，本地解析，yum源，known_hosts等
> (2) 拷贝软件安装包(存在content下命名为"\d{4}")到虚拟机，后续使用user_data脚本安装

___
#### Cloudinit结合ConfigDrive实现服务编排 ####
