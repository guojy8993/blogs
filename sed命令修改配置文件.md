#### 修改配置文件的配置 ####
```
[root@imageSRC testsh]# cat rsyncd.conf
auth users = root
address = 10.160.0.1223
```
```
[root@imageSRC testsh]# sed -ri "s/(address =).*/\1 10.160.0.144/"  rsyncd.conf
[root@imageSRC testsh]# sed -ri "s/(auth users =).*/\1 glance-admin/"  rsyncd.conf
```
```
[root@imageSRC testsh]# cat rsyncd.conf
auth users = glance-admin
address = 10.160.0.144
```

#### 修改配置文件删除指定行 ####

```
[root@dev ~]# cat /tmp/fstab 
/dev/rbd1 /mnt/volume123 xfs defaults,_netdev 0 0 
/dev/rbd2 /mnt/volume456 xfs defaults,_netdev 0 0
```
```
[root@dev ~]# sed -i "/.*volume456.*/d" /tmp/fstab
```
```
[root@dev ~]# cat /tmp/fstab 
/dev/rbd1 /mnt/volume123 xfs defaults,_netdev 0 0
```

