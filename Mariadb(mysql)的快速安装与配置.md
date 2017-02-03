1.安装mariadb

```
[root@master ~]# yum install mariadb mariadb-server MySQL-python
```

2.修改(添加)mariadb配置文件到  /etc/my.cnf.d/ 下,例如openstack的主控节点的数据库的配置,新加 mariadb_openstack.cnf文件并修改配置 

```
[root@master ~]# cat > /etc/my.cnf.d/mariadb_openstack.cnf << EOF
[mysqld]
bind-address=10.160.0.116
default-storage-engine=innodb
innodb_file_per_table
collation-server=utf8_general_ci
init-connect='SET NAMES utf8'
character-set-server=utf8
EOF
```

3.启动服务并设置开机自启

```
[root@master ~]# systemctl enable mariadb.service && systemctl start mariadb.service
```

4.安全初始化数据库
```
[root@master ~]#  mysql_secure_installation
NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.

Enter current password for root (enter for none):<无密码,点击回车键即可>
OK, successfully used password, moving on...

Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.

Set root password? [Y/n] Y
New password:<设置root密码>
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.
Remove anonymous users? [Y/n] Y     # 禁用匿名用户
 ... Success!
Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.
Disallow root login remotely? [Y/n] Y  # 禁用root远程登录
 ... Success!
By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.
Remove test database and access to it? [Y/n] Y  # 禁用test数据库及其访问设置
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!
Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.
Reload privilege tables now? [Y/n] Y   # 刷新权限
 ... Success!
Cleaning up...
All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.
Thanks for using MariaDB!
```

# 至此设置完毕
