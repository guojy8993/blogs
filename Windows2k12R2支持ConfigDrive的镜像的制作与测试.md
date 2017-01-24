### 本文档说明 ###

(1) 准备工作

(2) 启动虚拟机

(3)驱动的加载与安装

(4)防火墙的配置

(5)CloudBase的安装与配置

(6)镜像可用性的测试

(7)附录

___
#### 准备工作 ####

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

___
#### 启动虚拟机 ####

i. 确保服务器支持KVM虚拟化(环境配置略)

ii.新建两个linux bridge,保证后续制作的镜像的外内双网卡
```
[root@cs112-04 config-2]# brctl addbr br0 && ip link set br0 up
[root@cs112-04 config-2]# brctl addbr br1 && ip link set br1 up
```

iii.创建空系统盘,并使用libvirt.xml模版文件定义并启动虚拟机
```
[root@cs112-04 ~]# qemu-img create -f qcow2 /tmp/ws2012 15G
```

```
[root@cs112-04 config-2]# cat > /root/test.xml << EOF
<domain type='kvm' id='441'>
  <name>win2012</name>
  <uuid>34a3c1a4-b460-415c-bc2b-9783296a5b8d</uuid>
  <memory unit='KiB'>8388608</memory>
  <currentMemory unit='KiB'>8388608</currentMemory>
  <vcpu placement='static'>8</vcpu>
  <resource>
    <partition>/machine</partition>
  </resource>
  <os>
    <type arch='x86_64' machine='pc-i440fx-2.2'>hvm</type>
    <boot dev='cdrom'/>
    <boot dev='hd'/>
  </os>
  <features>
    <acpi/>
    <apic/>
  </features>
  <cpu mode='custom' match='exact'>
    <model fallback='allow'>Haswell-noTSX</model>
  </cpu>
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>restart</on_crash>
  <pm>
    <suspend-to-mem enabled='no'/>
    <suspend-to-disk enabled='no'/>
  </pm>
  <devices>
    <emulator>/usr/libexec/qemu-kvm</emulator>
    
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2' cache='writeback'/>
      <source file='/tmp/ws2012'/>
      <backingStore/>
      <target dev='vda' bus='virtio'/>
    </disk>
    
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/opt/win2012.iso'/>
      <target dev='hda' bus='ide'/>
      <readonly/>
    </disk>
    
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/opt/virtio.iso'/>
      <target dev='hdb' bus='ide'/>
      <readonly/>
    </disk>
    <disk type='file' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <source file='/opt/cloudinit.iso'/>
      <target dev='hdc' bus='ide'/>
      <readonly/>
    </disk>
    
    <controller type='usb' index='0' model='ich9-ehci1'>
      <alias name='usb'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x7'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci1'>
      <alias name='usb'/>
      <master startport='0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0' multifunction='on'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci2'>
      <alias name='usb'/>
      <master startport='2'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x1'/>
    </controller>
    <controller type='usb' index='0' model='ich9-uhci3'>
      <alias name='usb'/>
      <master startport='4'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x2'/>
    </controller>
    <controller type='pci' index='0' model='pci-root'>
      <alias name='pci.0'/>
    </controller>
    <controller type='ide' index='0'>
      <alias name='ide'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x01' function='0x1'/>
    </controller>
    <interface type='bridge'>
      <mac address='52:54:00:50:fc:80'/>
      <source bridge='br0'/>
      <target dev='vnet0'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
    </interface>
    <interface type='bridge'>
      <mac address='52:54:00:1e:9c:7e'/>
      <source bridge='br1'/>
      <target dev='vnet1'/>
      <model type='virtio'/>
      <alias name='net1'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x06' function='0x0'/>
    </interface>

    <serial type='file'>
      <source path='/tmp/console.log'/>
      <target port='0'/>
    </serial>
    <serial type='pty'>
      <target port='1'/>
    </serial>
    <console type='file'>
      <source path='/tmp/console.log'/>
      <target type='serial' port='0'/>
    </console>

    <input type='tablet' bus='usb'>
      <alias name='input0'/>
    </input>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <graphics type='vnc' port='5901' autoport='yes' listen='0.0.0.0'>
      <listen type='address' address='0.0.0.0'/>
    </graphics>
    <video>
      <model type='cirrus' vram='16384' heads='1'/>
      <alias name='video0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x02' function='0x0'/>
    </video>
    <memballoon model='virtio'>
      <alias name='balloon0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </memballoon>
  </devices>
</domain>
```

```
[root@cs112-04 config-2]# virsh define /root/test.xml 
Domain win2012 defined from /root/test.xml
```

```
[root@cs112-04 config-2]# virsh start win2012
Domain win2012 started

```

此时虚拟机所有必需的ISO盘都具备:

```
[root@cs112-04 ~]# virsh domblklist win2012
Target     Source
------------------------------------------------
vda        /tmp/ws2012
hda        /opt/win2012.iso
hdb        /opt/virtio.iso
hdc        /opt/cloudinit.iso
```

使用vncviewer进入安装界面

![选择本地语言时间货币](https://github.com/guojy8993/ImageCache/blob/master/set_local_001.jpg)

选择安装

![选择安装](https://github.com/guojy8993/ImageCache/blob/master/set_local_002.jpg)

选择安装系统版本

![选择安装系统版本](https://github.com/guojy8993/ImageCache/blob/master/set_local_003.jpg)

选择接受协议

![选择接受协议](https://github.com/guojy8993/ImageCache/blob/master/set_local_004.jpg)

找不到硬盘，需要加载硬盘驱动

![找不到硬盘，需要加载硬盘驱动](https://github.com/guojy8993/ImageCache/blob/master/set_local_005.jpg)

windows2012系统选择w7/amd64驱动

![windows系统选择w7/amd64驱动01](https://github.com/guojy8993/ImageCache/blob/master/set_local_006.jpg)
![windows系统选择w7/amd64驱动02](https://github.com/guojy8993/ImageCache/blob/master/set_local_007.jpg)
![windows系统选择w7/amd64驱动03](https://github.com/guojy8993/ImageCache/blob/master/set_local_008.jpg)


___
#### 驱动的加载与安装 ####

在 服务器管理器 > 工具 > 计算机管理 > 设备管理器 > 其他设备 中找到未安装驱动的设备

安装PCI驱动

![设备管理器](https://github.com/guojy8993/ImageCache/blob/master/set_local_009.jpg)
![PCI设备驱动](https://github.com/guojy8993/ImageCache/blob/master/set_local_010.jpg)
![PCI设备驱动](https://github.com/guojy8993/ImageCache/blob/master/set_local_011.jpg)

安装以太网控制器驱动(两个网卡均如下设置)

![安装以太网控制器驱动](https://github.com/guojy8993/ImageCache/blob/master/set_local_012.jpg)


___
#### 防火墙的配置 ####
设置防火墙：允许ping以及RDP远程端口访问

![允许ping](https://github.com/guojy8993/ImageCache/blob/master/set_local_013.jpg)
![允许RDP远程端口访问](https://github.com/guojy8993/ImageCache/blob/master/set_local_014.jpg)

设置powershell安全策略：允许自定义PS脚本的运行

![允许自定义PS脚本的运行](https://github.com/guojy8993/ImageCache/blob/master/set_local_015.jpg)

___
#### CloudBase的安装与配置 ####

安装CloudBase

![允许自定义PS脚本的运行](https://github.com/guojy8993/ImageCache/blob/master/set_local_016.jpg)

设置username以及系统日志输出串口

![设置username以及系统日志输出串口](https://github.com/guojy8993/ImageCache/blob/master/set_local_017.jpg)

在sysprep之前修改CloudBase配置

![修改CloudBase配置01](https://github.com/guojy8993/ImageCache/blob/master/set_local_018.jpg)
![修改CloudBase配置02](https://github.com/guojy8993/ImageCache/blob/master/set_local_019.jpg)


继续执行Sysprep

![执行Sysprep1](https://github.com/guojy8993/ImageCache/blob/master/sysprep.jpg)
![执行Sysprep2](https://github.com/guojy8993/ImageCache/blob/master/syspreping.jpg)

然后等模板虚拟机关机，在宿主上将模板拷贝予以备份

```
[root@cs112-04 ~]# cp /tmp/ws2012 /opt/ws2012.backup 
```


___
#### 镜像可用性的测试 ####

在线编辑虚拟机配置,设置hd优先启动，并删除仅留一个hda,且设置hda为空设备

```
[root@cs112-04 ~]# virsh edit win2012
...
```
```
[root@cs112-04 ~]# virsh domblklist win2012
Target     Source
------------------------------------------------
vda        /tmp/ws2012
hda        -
```

制作ConfigDrive: 生成config drive 文件树
```
[root@cs112-04 ~]# cd /root/config-2/
[root@cs112-04 config-2]# tree
.
└── openstack
    ├── content
    │   ├── 0000
    │   └── 0001
    └── latest
        ├── meta_data.json
        ├── user_data
        └── vendor_data.json
3 directories, 5 files
```
```
[root@cs112-04 config-2]# cat openstack/content/0000
# powershell
```
```
[root@cs112-04 config-2]# cat openstack/content/0001
TYPE=Ethernet
BOOTPROTO=static
DEVICE=eth0
ONBOOT=yes
MTU=1500
IPADDR=122.111.126.212
PREFIX=24
GATEWAY=122.111.126.1
DNS1=8.8.8.8
DNS2=8.8.4.4
```

```
[root@cs112-04 config-2]# cat openstack/latest/meta_data.json 
{"files": [{"path": "C:/", "content_path": "/content/0000"},
{"path": "C:/", "content_path": "/content/0001"}],
"admin_pass": "guojy8993@163!", "hostname": "demo.cloud.org", 
"launch_index": 0, "name": "demo", "uuid": "8c83e8c0-2911-48ea-be13-fdfd18389f04"}
```

```
[root@cs112-04 config-2]# cat openstack/latest/user_data 
#ps1_sysnative
echo 1qazxsw2# > c:\passwd.txt
net user administrator 1qazxsw2#
$vifs = netsh interface ipv4 show interfaces
$eth0_index = $vifs[-3].split()[1]
$eth1_index = $vifs[-2].split()[1]
netsh interface ipv4 add address $eth0_index 122.111.126.5   255.255.255.0 122.111.126.1
netsh interface ipv4 add dnsservers $eth0_index 8.8.8.8
netsh interface ipv4 add dnsservers $eth0_index 8.8.4.4 index=2
netsh interface ipv4 add address $eth1_index 10.100.0.100    255.255.0.0   10.100.0.1
echo "Network Configuration Done !" > c:\passwd.txt
```
```
[root@cs112-04 config-2]# cat openstack/latest/vendor_data.json 
{}
```

制作ConfigDrive: 生成config drive光盘文件
```
[root@cs112-04 config-2]# mkisofs -l -J -L -r -V config-2 -o /root/haproxy.iso .
```

冷挂载光盘文件到虚拟机(经过sysprep虚拟机已经关机),并启动.

```
[root@cs112-04 config-2]# virsh change-media win2012 hda /root/haproxy.iso --config 
Successfully updated media.

[root@cs112-04 config-2]# virsh start win2012
Domain win2012 started
```

等待虚拟机进行配置注入 ...

使用metadata注入的密码登录，修改新密码，查看虚拟机的配置信息，并进行网络测试

![查看虚拟机的配置信息](https://github.com/guojy8993/ImageCache/blob/master/hostname_ip.jpg)
![网络测试](https://github.com/guojy8993/ImageCache/blob/master/ping.jpg)

#### 附录 ####

[1.OpenStack官方镜像制作](http://docs.openstack.org/image-guide/windows-image.html)

[2.UserData支持powershell](https://github.com/openstack/cloudbase-init/blob/master/doc/source/userdata.rst)

[3.CloudBasae-init提供的插件与服务](https://github.com/openstack/cloudbase-init/blob/master/doc/source/plugins.rst)

[4.Windows2k12设置默认允许Ping](http://ms.n.blog.163.com/blog/static/18595352014112384724884/)

[5.CloudBase官方下载](http://www.cloudbase.it/downloads/CloudbaseInitSetup_Stable_x64.msi)

[6.VirtIO驱动官方下载](https://fedoraproject.org/wiki/Windows_Virtio_Drivers#Direct_download)

[7. 如何使用 mkisofs 制作镜像](https://github.com/guojy8993/blogs/blob/master/linux%E4%B8%8B%E7%9A%84ISO%E7%94%9F%E6%88%90%E5%B7%A5%E5%85%B7%E4%B8%8E%E4%BD%BF%E7%94%A8.md)
