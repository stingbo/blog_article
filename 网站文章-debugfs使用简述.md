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

1. 根据块号查找分区。查看分区情况，`sudo fdisk -lu`
	```
    Device         Start       End   Sectors   Size Type
    /dev/sda1       2048  99999743  99997696  47.7G Linux filesystem
    /dev/sda2   99999744 100999167    999424   488M Linux filesystem
    /dev/sda3  100999168 117000191  16001024   7.6G Linux swap
    /dev/sda4  117000192 119001087   2000896   977M EFI System
    /dev/sda5  119001088 500117503 381116416 181.7G Linux filesystem  # End - Start = Sectors
    ```
	由`Block(4KB) = 8 * Sectors(0.5KB)`与`b = (int)((L-S)*512/B)`
    > where:
    b = File System block number
    L = LBA(Logical Block Address)
    S = Starting sector of partition as shown by fdisk -lu and (int) denotes the integer part
    B = File system block size in bytes

    可知： `L-S = b / (512 / B) = 43769344 / (512 / 4096) = 350154752`。所以， `L = 350154752 + 119001088('注:此值为/dev/sda5 Start位置') = 469155840`，判断这个块号应该是在/dev/sda5里，在Start到End区间内(如果判断不了，使用下一步试一下也能知道)。

2. 使用debugfs获取对应具体文件信息，`sudo debugfs /dev/sda5`

    ```
    [sting@sting-snds]
    [:~]$ sudo debugfs /dev/sda5
    [sudo] password for sting:
    debugfs 1.42.13 (17-May-2015)
    debugfs:  icheck 43769344  #如果不在这个分区，会提示 <block not found>
    Block	Inode number
    43769344	10485770
    debugfs:  ncheck 10485770
    Inode	Pathname
    10485770	/sting/.Xauthority # 文件路径
    debugfs:  q
    ```