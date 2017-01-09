### 文档内容提要 ###
1.使用 fdisk 磁盘分区

2.使用 parted 磁盘分区

#### 使用 fdisk 磁盘分区 ####
```
[root@frontend ~]# fdisk /dev/sdb 
Welcome to fdisk (util-linux 2.23.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-104857599, default 2048): 
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-104857599, default 104857599): +10G
Partition 1 of type Linux and of size 10 GiB is set

Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
[root@frontend ~]# fdisk -l | grep sdb
Disk /dev/sdb: 53.7 GB, 53687091200 bytes, 104857600 sectors
/dev/sdb1            2048    20973567    10485760   83  Linux
```
_ _ _

#### 使用 parted进行 磁盘分区 ####
脚本实现如下:
```
#!/bin/bash
# Author  guojy
# Date    2016/09/26
# Email   guojy8993@163.com

partion_max=5
capacity_gb_per_partion=10
device=/dev/sdb
fs=xfs

function clear_partions()
{
    for partion_id in `parted ${device} print | egrep -v \
                  "Number|Model|Disk|Sector|Partition|^$" | awk '{print $1}'`; 
    do
        parted ${device} rm ${partion_id} &>/dev/null;
    done
}
function set_gpt_partion_table()
{
    echo "yes\r" | parted ${device} mklabel  gpt     
}
function make_partions()
{
    for i in $(seq 1 ${partion_max});
    do
        if [ ${i} -eq 1 ];then
            start=0
            end=${capacity_gb_per_partion}
        else
            end=$(( capacity_gb_per_partion * i ))
            start=$(( end - capacity_gb_per_partion ))
        fi
        parted ${device} mkpart primary ${fs} ${start}G ${end}G &>/dev/null       
    done
    parted ${device} print
}

#######

clear_partions
set_gpt_partion_table
make_partions
```

说明:

gpt+parted分区方式相对fdisk有诸多优点:

(1)主分区个数充分满足需求(扩展分区就没必要了) 

(2)一条命令即可分出一个分区(免交互,对于自动化运维很有用. 前提是设置分区起始sector满足对齐,必须系统提示!)

(3)gpt可以创建出2T+的分区,fdisk是不可以的.

(4)gpt提供分区表的冗余以实现分区表的备份与安全
