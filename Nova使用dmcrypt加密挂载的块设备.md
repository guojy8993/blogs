### 利用dm-crypt来创建加密文件系统 ###
(1) 前言

(2) 实践

(3) 总结

#### 前言 ####
```
    Linux使用加密文件系统后，数据的安全能得到很好的保护。在这种情况下，即使把我们的机器送给黑客，只要他们没有密钥，黑客看到的数
据只会是一堆乱码，毫无利用价值可言.
    本文将详细介绍利用dm-crypt来创建加密文件系统的方法。与其它创建加密文件系统的方法相比，dm-crypt系统有着无可比拟的优越性：
它的速度更快，易用性更强。除此之外，它的适用面也很广，能够运行在各种块设备上，即使这些设备使用了RAID和LVM也毫无障碍. dm-crypt
系统之所以具有这些优点，主要得益于该技术是建立在2.6版本内核的device-mapper特性之上的. device-mapper是设计用来为在实际的块设
备之上添加虚拟层提供一种通用灵活的方法，以方便开发人员实现镜像、快照、级联和加密等处理。此外，dm-crypt使用了内核密码应用编程接
口实现了透明的加密，并且兼容cryptloop系统.
```

#### 实践 ####
(1) 配置内核
```
    dm-crypt利用内核的密码应用编程接口来完成密码操作。一般说来，内核通常将各种加密程序以模块的形式加载。对于256-bit AES来说，
其安全强度已经非常之高，即便用来保护绝密级的数据也足够了。因此本文中我们使用256-bit AES密码，为了保证您的内核已经加载AES密码模
块，请利用下列命令进行检查：
[root@dev ~]# cat /proc/crypto
　　如果看到类似下面的输出的话，说明AES模块已经加载： 
　　name : aes
　　module : aes
　　type : cipher
　　blocksize : 16
　　min keysize : 16
　　max keysize : 32
　　否则，我们可以利用modprobe来手工加载AES模块，命令如下所示：
　　$ sudo modprobe aes
　　接下来安装dmsetup软件包，该软件包含有配置device-mapper所需的工具：
　　$ sudo apt-get install dmsetup cryptsetup
　　为检查dmsetup软件包是否已经建立了设备映象程序，键入下列命令： 
　　$ ls -l /dev/mapper/control
　　接下来加载dm-crypt内核模块： 
　　$ sudo modprobe dm-crypt
　　dm-crypt加载后，它会用evice-mapper自动注册。如果再次检验的话，device-mapper已能识别dm-crypt，并且把crypt 添加为可用的对象： 
　　$ sudo dmsetup targets
　　如果一切顺利，现在你应该看到crypt的下列输出： 
　　crypt v1.1.0
　　striped v1.0.2
　　linear v1.0.1
　　error v1.0.1
　　这说明我们的系统已经为装载加密设备做好了准备。下面，我们先来建立一个加密设备。
　```　

　　二、建立加密设备
　　要创建作为加密设备装载的文件系统，有两种选择：一是建立一个磁盘映像，然后作为回送设备加载；二是使用物理设备。无论那种情况，除了在建立和捆绑回送设备外，其它操作过程都是相似的。
　　1.建立回送磁盘映象
　　如果你没有用来加密的物理设备（比如存储棒或另外的磁盘分区），作为替换，你可以利用命令dd来建立一个空磁盘映象，然后将该映象作为回送设备来装载，照样能用。下面我们以实例来加以介绍：
　　$ dd if=/dev/zero of=~/secret.img bs=1M count=100
　　这里我们新建了一个大小为100 MB的磁盘映象，该映象名字为secret.img。要想改变其大小，可以改变count的值。 
　　接下来，我们利用losetup命令将该映象和一个回送设备联系起来： 
　　$ sudo losetup /dev/loop/0 ~/secret.img
　　现在，我们已经得到了一个虚拟的块设备，其位于/dev/loop/0，并且我们能够如同使用其它设备那样来使用它。 
　　2.设置块设备
　　准备好了物理块设备（例如/dev/sda1），或者是虚拟块设备（像前面那样建立了回送映象，并利用device-mapper将其作为加密的逻辑卷加载），我们就可以进行块设备配置了。
　　下面我们使用cryptsetup来建立逻辑卷，并将其与块设备捆绑：
　　$ sudo cryptsetup -y create myEncryptedFilesystem 
　　/dev/DEVICENAME
　　其中，myEncryptedFilesystem 是新建的逻辑卷的名称。并且最后一个参数必须是将用作加密卷的块设备。所以，如果你要使用前面建立的回送映象作为虚拟块设备的话，应当运行以下命令： 
　　$ sudo cryptsetup -y create myEncryptedFilesystem /dev/loop/0
　　无论是使用物理块设备还是虚拟块设备，程序都会要你输入逻辑卷的口令，-y的作用在于要你输入两次口令以确保无误。这一点很重要，因为一旦口令弄错，你就会把自己的数据锁住，这时谁也帮不了您了！
　　为了确认逻辑卷是否已经建立，可以使用下列命令进行检查一下：
　　$ sudo dmsetup ls
　　只要该命令列出了逻辑卷，就说明已经成功建立了逻辑卷。不过根据机器的不同，设备号可能有所不同： 
　　myEncryptedFilesystem (221, 0)
　　device-mapper会把它的虚拟设备装载到/dev/mapper下面，所以，你的虚拟块设备应该是/dev/mapper/myEncryptedFilesystem ，尽管用起来它和其它块设备没什么不同，实际上它却是经过透明加密的。
　　如同物理设备一样，我们也可以在虚拟设备上创建文件系统：
　　$ sudo mkfs.ext3 /dev/mapper/myEncryptedFilesystem
　　现在为新的虚拟块设备建立一个装载点，然后将其装载。命令如下所示： 
　　$ sudo mkdir /mnt/myEncryptedFilesystem 
　　$ sudo mount /dev/mapper/myEncryptedFilesystem /mnt/myEncryptedFilesystem
　　我们能够利用下面的命令查看其装载后的情况： 
　　$ df -h /mnt/myEncryptedFilesystem 
　　Filesystem Size Used Avail Use% Mounted on
　　/dev/mapper/myEncryptedFilesystem 97M 2.1M 90M 2% /mnt/myEncryptedFilesystem
　　很好，我们看到装载的文件系统，尽管看起来与其它文件系统无异，但实际上写到/mnt/myEncryptedFilesystem /下的所有数据，在数据写入之前都是经过透明的加密处理后才写入磁盘的，因此，从该处读取的数据都是些密文。
　　

　　要卸载加密文件系统，和平常的方法没什么两样：
　　$ sudo umount /mnt/myEncryptedFilesystem
　　即便已经卸载了块设备，在dm-crypt中仍然视为一个虚拟设备。如若不信，你可以再次运行命令sudo dmsetup ls来验证一下，你会看到该设备依然会被列出。因为dm-crypt缓存了口令，所以机器上的其它用户不需要知道口令就能重新装载该设备。为了避免这种情况发生，你必须在卸载设备后从dm-crypt中显式的删除该设备。命令具体如下所示： 
　　$ sudo cryptsetup remove myEncryptedFilesystem
　　此后，它将彻底清除，要想再次装载的话，你必须再次输入口令。为了简化该过程，我们可以利用一个简单的脚本来完成卸载和清除工作： 
　　#!/bin/sh
　　umount /mnt/myEncryptedFilesystem 
　　cryptsetup remove myEncryptedFilesystem
　　四、重新装载
　　在卸载加密设备后，我们很可能还需作为普通用户来装载它们。为了简化该工作，我们需要在/etc/fstab文件中添加下列内容： 
　　/dev/mapper/myEncryptedFilesystem /mnt/myEncryptedFilesystem ext3 noauto,noatime 0 0
　　此外，我们也可以通过建立脚本来替我们完成dm-crypt设备的创建和卷的装载工作，方法是用实际设备的名称或文件路径来替换/dev/DEVICENAME： 
　　#!/bin/sh
　　cryptsetup create myEncryptedFilesystem /dev/DEVICENAME
　　mount /dev/mapper/myEncryptedFilesystem /mnt/myEncryptedFilesystem
　　如果你使用的是回送设备的话，你还能利用脚本来捆绑设备： 
　　#!/bin/sh 
　　losetup /dev/loop/0 ~/secret.img
　　cryptsetup create myEncryptedFilesystem /dev/loop/0
　　mount /dev/mapper/myEncryptedFilesystem /mnt/myEncryptedFilesystem
　　如果你收到消息“ioctl: LOOP_SET_FD: Device or resource busy”，这说明回送设备很可能仍然装载在系统上。我们可以利用sudo losetup -d /dev/loop/0命令将其删除。
　　五、加密主目录
　　如果配置了PAM（Pluggable Authentication Modules，即可插入式鉴别模块）子系统在您登录时装载主目录的话，你甚至还能加密整个主目录。因为libpam-mount模块允许PAM在用户登录时自动装载任意设备，所以我们要连同openssl一起来安装该模块。命令如下所示：
　　$ sudo apt-get install libpam-mount openssl
　　接下来，编辑文件/etc/pam.d/common-auth，在其末尾添加下列一行： 
　　auth optional pam_mount.so use_first_pass
　　然后在文件/etc/pam.d/common-session末尾添加下列一行内容： 
　　session optional pam_mount.so
　　现在，我们来设置PAM，告诉它需要装载哪些卷、以及装载位置。对本例而言，假设用户名是Ian，要用到的设备是/dev/sda1，要添加到/etc/security/pam_mount.conf文件中的内容如下所示：
　　volume Ian crypt - /dev/sda1 /home/Ian cipher=aes aes-256-ecb /home/Ian.key
　　如果想使用磁盘映象，你需要在此规定回送设备（比如/dev/loop/0），并确保在Ian登录之前系统已经运行losetup。为此，你可以将 losetup /dev/loop/0 /home/secret.img放入/etc/rc.local文件中。因为该卷被加密，所以PAM需要密钥来装载卷。最后的参数用来告诉PAM密钥在 /home/Ian.key文件中，为此，通过使用OpenSSL来加密你的口令来建立密钥文件：
　　$ sudo sh -c 'echo 
　　'
　　YOUR PASSPHRASE
　　' 
　　| openssl aes-256-ecb >
　　/home/Ian.key'
　　这时，提示你输入密码。注意，这里的口令必需和想要的用户登录密码一致。原因是当你登录时，PAM需要你提供这个密码，用以加密你的密钥文件，然后根据包含在密钥文件中的口令用dm-crypt装载你的主目录。
　　需要注意的是，这样做会把你的口令以明文的形式暴露在.history文件中，所以要及时利用命令history -c清楚你的历史记录。此外，要想避免把口令存放在加密的密钥文件中的话，可以让创建加密文件系统的口令和登录口令完全一致。这样，在身份认证时，PAM 只要把你的密码传给dm-crypt就可以了，而不必从密钥文件中抽取密码。为此，你可以在/etc/security/pam_mount.conf文件中使用下面的命令行：
　　volume Ian crypt - /dev/sda1 /home/Ian cipher=aes - -
　　最后，为了保证在退出系统时自动卸载加密主目录，请编辑/etc/login.defs文件使得CLOSE_SESSIONS项配置如下： 
　　CLOSE_SESSIONS yes

#### 总结 ####
    数据加密是一种强而有力的安全手段，它能在各种环境下很好的保护数据的机密性。而本文介绍的Ubuntu Linux 下的加密文
件系统就是一种非常有用的数据加密保护方式，相信它能够在保护数据机密性相方面对您有所帮助.
