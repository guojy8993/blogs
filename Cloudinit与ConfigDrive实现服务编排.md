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
虽然Cloud-Init官方文档以及OpenStack相关文档介绍了ConfigDrive内部复杂的文件系统组织以及数据结构,但结合笔者测试,归结了
最简的文件结构,参考下文:
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
___
#### Cloudinit结合ConfigDrive实现服务编排 ####
