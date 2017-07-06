##### 关于Linux交换分区
		Linux RAM由内存页组成的块组成的。为了释放RAM，就可能产生“linux交换分区”，并将部分内存从RAM复制到硬盘上的预配置空间。Linux交换分区就是为了允许系统利用比原来可用的内存更多的内存。
		但是，交换分区也有一些缺点。由于硬盘的读写速度比RAM慢得多，因此服务器性能可能会大大减慢。另外，如果系统有太多文件从交换分区写入和读出，那么swap thrashing可能发生(这会导致效率低下，因为大部分时间消耗在访问磁盘上)。

##### 增加交换空间步骤：
* 检查交换分区空间
在我们继续交换分区文件之前，我们需要查看交换分区使用情况，检查是否已经启用了交换分区文件。
命令：`swapon -s`
如果没有返回，则摘要为空，不存在交换分区文件。

* 检查文件系统
在知道我们没有启用交换分区文件后，我们可以使用df命令检查服务器上有多少剩余空间。交换分区文件需要512MB，以下结果显示我们只使用了大约7％的`/dev/hda`磁盘空间。
```
df
Filesystem           1K-blocks      Used Available Use% Mounted on
/dev/hda              20642428   1347968  18245884   7% /
```

* 创建并启用交换分区文件
现在是使用dd命令创建交换分区文件本身的时候了：
`sudo dd if=/dev/zero of=/swapfile bs=1024 count=512k`
“of=/swapfile”指定文件的名称，这里我们取的名称是swapfile。

* 为交换文件创建一个linux交换分区区域：
`sudo mkswap /swapfile`
结果显示：
```
Setting up swapspace version 1, size = 536866 kB
```

* 通过激活交换分区文件完成工作：
`sudo swapon /swapfile`

* 查看交换分区摘要，将可以看到新的交换分区文件。
`swapon -s`
```
swapon -s
Filename				Type		Size	Used	Priority
/swapfile                               file		524280	0	-1
```

* 做完这些工作后，如果服务器重启，那么这个交换分区会消失。可以通过将交换分区配置添加到fstab文件来确保交换分区一直存在。
打开文件：
`sudo vim /etc/fstab`
在fstab文件最下面粘贴在以下内容就可以：
```
/ swapfile swap swap defaults 0 0
```

* 这个交换分区文件只应该有读的权限，所以我们要给它设置正确的权限：
`chown root:root /swapfile`
`chmod 0600 /swapfile`

##### 如何配置Swappiness
操作系统内核可以通过称为`swappiness`的配置参数来调整依赖交换分区的使用频率。

* 要查看当前的swappiness设置，使用命令：
`cat /proc/sys/vm/swappiness`
结果显示为：
```
60
```

> Swapiness是0到100之间的值。接近100的Swappiness意味着操作系统会频繁的使用交换分区。虽然交换分区给系统提供额外的资源，但RAM比交换分区空间读写速度快得多。不管什么时候，程序从RAM移动到交换分区的话，它的运行速度都会减慢。
当Swappiness值为0时意味着操作只会在必须需要依靠交换分区时才会使用到它，比如，不使用交换分区，内存会溢出。

* 我们可以使用sysctl命令调整swappiness：
`sysctl vm.swappiness = 10`
```
vm.swappiness = 10
```

* 如果我们再次检查系统的swappiness值，可以看到已经修改了：
`cat /proc/sys/vm/swappiness`
```
10
```

要使您的服务器每次启动时自动应用此设置，您可以将该设置添加到`/etc/sysctl.conf`文件中：
`sudo vim /etc/sysctl.conf`
增加内容： `vm.swappiness = 10`