1.创建补丁

补丁生成命令模版: diff -uN {old_version_file}  {new_version_file} > {patch_file}

应用场景将 v1(2016-06-06)的instance.py 升级到v2(2016-07-14)的 instance.py

首先将最新文件与老版本文件对比,生成补丁
```
[root@agent144 ~]# diff -uN instance.py_20160616 instance.py_20160714 > patch_instance_20160714
```

2.应用补丁

应用补丁的命令模版: patch -p 0 {old_version_file}   {patch_file}

应用场景将 v2(20160714)补丁应用到的v1(20160606)的instance.py

```
[root@agent144 ~]# patch -p 0 instance.py_20160616 patch_instance_20160714
[root@agent144 ~]# diff -uN instance.py_20160616 instance.py_20160714 
[root@agent144 ~]#
```
应用补丁之后确定升级到文件指定版本

3.反应用补丁

应用补丁以实现文件版本回退的命令模版: patch -R -i {patch_file} {patched_old_version_file}

如果发现新版本有问题，回退版本到上一版

```
[root@agent144 ~]# patch -R -i patch_instance_20160714 instance.py_20160616
```
