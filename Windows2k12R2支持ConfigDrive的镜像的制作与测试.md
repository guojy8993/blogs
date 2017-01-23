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
```
[root@cs112-04 config-2]# brctl addbr br0 && ip link set br0 up
[root@cs112-04 config-2]# brctl addbr br1 && ip link set br1 up
```

iii.创建空系统盘,并使用virt-install启动虚拟机
```
[root@cs112-04 ~]# qemu-img create -f qcow2 /tmp/ws2012 15G
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

[root@cs112-04 config-2]# virsh define /root/test.xml 
Domain win2012 defined from /root/test.xml

[root@cs112-04 config-2]# virsh start win2012
Domain win2012 started

```

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
