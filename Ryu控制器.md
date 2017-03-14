'''
[root@dev ~]# ovs-ofctl  add-flow br-lan "dl_type=0x88cc,dl_dst=01:80:c2:00:00:0e,in_port=1 actions=DROP"

{
    "actions": [
        {"type":"OUTPUT"}
    ],
    "dpid":"6762166805825",
    "priority": 100,
    "table_id": 0,
    "match": {
        "dl_type": 35020,
        "dl_dst": "01:80:c2:00:00:03",
        "in_port": 1
    }
}
'''
### 附录 ###
[链路层协议类型知多少](http://www.2cto.com/net/201208/151809.html)
[lldp协议详解](http://blog.csdn.net/goodluckwhh/article/details/10948065)
[MIB是什么?](http://baike.baidu.com/item/mib/4490795)
[如何查看MIB信息?](http://blog.chinaunix.net/uid-28813320-id-5748106.html)
[CentOS配置lldp与交换机建立邻居关系](http://blog.csdn.net/achejq/article/details/52062240)
[Layer2的保留地址](http://blog.csdn.net/shanzhizi/article/details/7750538)

[RYU控制器](http://www.sdnlab.com/1785.html)
[RYU官方文档](http://ryu.readthedocs.io/en/latest/developing.html)
[RYU系统架构详解](http://osrg.github.io/ryu-book/en/html/)
[SDNLAB大神的博文](http://www.sdnlab.com/author/9/page/4/)
[OpenVswitch常用命令](http://blog.csdn.net/tantexian/article/details/46707175)
