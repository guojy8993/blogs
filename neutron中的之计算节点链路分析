
1.计算节点虚拟机实例信息如下:
----------------------------------------------------------------------------------------------
NAME       NETWORK(fixed_ip,floating_ip)                   fixed_ip_mac       device
hyp-vm1    net-hyp=192.168.66.2          122.164.124.21    fa:16:3e:23:91:7b  tap156d4a84-8f
hyp-vm2    net-hyp=192.168.66.4          -                 fa:16:3e:61:d8:b7  tap9b3f4311-42
liyang-vm  net-liyang0993=192.168.100.3  122.164.124.45    fa:16:3e:6c:c9:bb  tap0620d5f8-02
----------------------------------------------------------------------------------------------
注: 虚拟机ip与网卡信息可以从dashboard以及计算节点的virsh管理工具收集,此处略.

2.虚拟机直连 linux bridge,每个虚拟机拥有独立的qbr.
[root@compute home]# brctl show
bridge name    bridge id         STP enabled  interfaces
qbr0620d5f8-02 8000.e2ac0a53cb2b no           qvb0620d5f8-02  tap0620d5f8-02
qbr156d4a84-8f 8000.62f2e45acb48 no           qvb156d4a84-8f  tap156d4a84-8f
qbr9b3f4311-42 8000.9ea9c59ddd0b no           qvb9b3f4311-42  tap9b3f4311-42

注: qbr是neutron安全组具体实现的地方,留待专题说明

3.qbr网桥上的qvb设备与br-int上某对应qvo设备以对等设备的形式直连
[root@compute home]# ovs-vsctl show
1db94f48-8307-4bad-b109-30897062c4e4
    """br-tun 信息略"""
    Bridge br-int
        fail_mode: secure
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port br-int
            Interface br-int
                type: internal
        Port "qvo156d4a84-8f"
            tag: 2
            Interface "qvo156d4a84-8f"
        Port "qvo9b3f4311-42"
            tag: 1
            Interface "qvo9b3f4311-42"
        Port "qvo0620d5f8-02"
            tag: 2
            Interface "qvo0620d5f8-02"
    ovs_version: "2.4.0"
