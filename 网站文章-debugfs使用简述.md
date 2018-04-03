##### 知道具体文件，查看文件所存放的block号：
1. 比如我现在要查看/home/sting/.Xauthority文件所占的block号，`df -h` 查看分区对应的系统目录
    ```
    Filesystem      Size  Used Avail Use% Mounted on
    udev            3.9G     0  3.9G   0% /dev
    tmpfs           789M   42M  748M   6% /run
    /dev/sda1        47G   32G   13G  72% /
    tmpfs           3.9G  423M  3.5G  11% /dev/shm
    tmpfs           5.0M  4.0K  5.0M   1% /run/lock
    tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
    /dev/loop2      207M  207M     0 100% /snap/ubuntu-app-platform/34
    /dev/loop1       82M   82M     0 100% /snap/core/4206
    /dev/loop0       82M   82M     0 100% /snap/core/4017
    /dev/loop4       62M   62M     0 100% /snap/go/1473
    /dev/loop3       82M   82M     0 100% /snap/core/4110
    /dev/loop5       11M   11M     0 100% /snap/ipfs/335
    /dev/sda2       465M  396M   41M  91% /boot
    /dev/sda5       179G   59G  112G  35% /home
    /dev/sda4       976M  3.5M  972M   1% /boot/efi
    tmpfs           789M  140K  789M   1% /run/user/1000
    ```

2. `sudo debugfs /dev/sda5`

    ```
    [sting@sting-snds]
    [:~]$ sudo debugfs /dev/sda5
    [sudo] password for sting:
    debugfs 1.42.13 (17-May-2015)
    debugfs:  pwd
    [pwd]   INODE:      2  PATH: /    # 由上一步所知，这时你所在的目录其实在/home下
    [root]  INODE:      2  PATH: /
    debugfs:  stat /sting/.Xauthority # 查看这个文件的信息，它的实际路径其实是 /home/sting/.Xauthority
    	# 显示如下：
        Inode: 10485770   Type: regular    Mode:  0600   Flags: 0x80000
        Generation: 2100926769    Version: 0x00000000:00000001
        User:  1000   Group:  1000   Size: 55
        File ACL: 0    Directory ACL: 0
        Links: 1   Blockcount: 8
        Fragment:  Address: 0    Number: 0    Size: 0
         ctime: 0x5ab84844:5b09ecdc -- Mon Mar 26 09:09:24 2018
         atime: 0x5ac2db93:a9a86890 -- Tue Apr  3 09:40:35 2018
         mtime: 0x5ab84844:5b09ecdc -- Mon Mar 26 09:09:24 2018
        crtime: 0x599bcd3f:1183b45c -- Tue Aug 22 14:20:47 2017
        Size of extra inode fields: 32
        EXTENTS:
        (0):43769344  # 文件对应的块号
        (END)
    ```

##### 知道block号，查看block对应的具体文件：

1. 查看文件路径
    ```
    [sting@sting-snds]
    [:~]$ sudo debugfs /dev/sda5
    [sudo] password for sting:
    debugfs 1.42.13 (17-May-2015)
    debugfs:  icheck 43769344
    Block	Inode number
    43769344	10485770
    debugfs:  ncheck 10485770
    Inode	Pathname
    10485770	/sting/.Xauthority # 文件路径
    debugfs:  q
    ```