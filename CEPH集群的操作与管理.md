### 文章内容提要
- CEPH集群缩容

#### CEPH集群缩容
应用场景: 随着业务数量的减少,需要缩容存储集群，释放出某存储节点
说明: 本章节以移除测试环境的ceph集群的node-2节点为例予以说明,测试环境情况如下
```
[root@node-1 ~]# ceph osd tree
ID WEIGHT  TYPE NAME       UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1 0.14456 root default                                      
-2 0.14456     host node-2                                   
 0 0.04819         osd.0        up  1.00000          1.00000 
 1 0.04819         osd.1        up  1.00000          1.00000 
 2 0.04819         osd.2        up  1.00000          1.00000
```
缩容集群一般遵循如下思路:
1. 标记目标osd为out状态使之触发数据迁出
说明: osd标记为out状态之后,ceph将会立即开始数据重新平衡,它会将osd上的pg迁移到集群内部其他的osd上。在一定
时间内，集群处于不健康状态，但是它还是能够对客户端提供服务。根据被移除的osd的数目，系统的性能在数据恢复结
束之前可能有一定程度的下降。一旦集群恢复健康状态，他就会像往常一样运行了。

[root@node-1 ~]# ceph osd out osd.0
marked out osd.0. 
[root@node-1 ~]# ceph osd tree
ID WEIGHT  TYPE NAME       UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1 0.14456 root default                                      
-2 0.14456     host node-2                                   
 0 0.04819         osd.0        up        0          1.00000 
 1 0.04819         osd.1        up  1.00000          1.00000 
 2 0.04819         osd.2        up  1.00000          1.00000


2. 查看集群状态等待其恢复完成
[root@node-1 ~]# ceph -s
    cluster 614b1692-ba57-4e3d-bccc-80b97a1f4c10
     health HEALTH_OK
     monmap e1: 1 mons at {node-1=10.120.3.2:6789/0}
            election epoch 12, quorum 0 node-1
     osdmap e88: 3 osds: 3 up, 2 in
            flags sortbitwise,require_jewel_osds
      pgmap v2231: 576 pgs, 5 pools, 97509 kB data, 31 objects
            20651 MB used, 80442 MB / 101094 MB avail
                 576 active+clean

3. 关闭目标osd对应的服务
说明: 
(1) 该操作需要切换到该osd所在的主机进行
(2) 关闭osd服务的方法版本不同略有变化

[root@node-1 ~]# ssh node-2
Warning: Permanently added 'node-2,10.120.1.3' (ECDSA) to the list of known hosts.
Last login: Fri Jul 28 06:56:48 2017 from 10.120.1.2
[root@node-2 ~]# systemctl stop ceph-osd@0.service

[root@node-1 ~]# ceph osd tree
ID WEIGHT  TYPE NAME       UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1 0.14456 root default                                      
-2 0.14456     host node-2                                   
 0 0.04819         osd.0      down        0          1.00000 
 1 0.04819         osd.1        up  1.00000          1.00000 
 2 0.04819         osd.2        up  1.00000          1.00000

4. 将目标osd从crush map中移除
[root@node-1 ~]# ceph osd crush remove osd.0
removed item id 0 name 'osd.0' from crush map

5. 将目标osd的验证密钥移除
[root@node-1 ~]# ceph auth del osd.0
updated

6. 最终删除目标osd
[root@node-1 ~]# ceph osd rm osd.0
removed osd.0



7. 如果需要从集群踢出主机，那么将该主机的全部osd按照步骤1-6执行全部移除，然后在crush map中移除该host bucket
[root@node-2 ~]# for i in {1..2};do \
> ceph osd out osd.${i}; \
> systemctl stop ceph-osd@${i}; \
> ceph osd crush remove osd.${i}; \
> ceph auth del osd.${i}; \
> ceph osd rm osd.${i}; \
> done
marked out osd.1. 
removed item id 1 name 'osd.1' from crush map
updated
removed osd.1
marked out osd.2. 
removed item id 2 name 'osd.2' from crush map
updated
removed osd.2

[root@node-2 ~]# ceph osd tree
ID WEIGHT TYPE NAME       UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1      0 root default                                      
-2      0     host node-2                                   
[root@node-2 ~]# ceph osd crush remove node-2
removed item id -2 name 'node-2' from crush map


