###### 本教程所用环境

    * 机器:普通电脑 ThinkPad E470c
    * 系统:Ubuntu 16.04 LTS

###### 步骤

* 下载元界钱包
	* 官方博客有一篇[元界(Metaverse)安装与使用手册](https://blog.mvs.live/mvs-user-guide-zh/)，这里面涵盖了Windows,MacOSX,Linux系统下钱包下载，安装，使用的详细说明，可能在一些细节与实际不同，摸索一下就会掌握，如有问题，欢迎在评论下留言，或者直接联系我。

* 下载挖矿软件
	1. 英语过关的请直接看这篇文章[Ethereum GPU Mining on Linux How-To](https://www.meebey.net/posts/ethereum_gpu_mining_on_linux_howto/),这里有如何下载以太币挖矿软件和显卡驱动。建议：最好结合我下面的一起操作 :) 。
	2. 我结合自己的实践来大概说明一下:
		* 用以下命令增加挖矿软件安装包的源：
        ```
        sudo apt-get install software-properties-common
        sudo add-apt-repository ppa:ethereum/ethereum
        sudo apt-get update
        ```
        如果是在`Debian 8`(在Ubuntu上你可以跳过此步骤)上你需要使用以下命令替换源名称：
        ```
        sudo sed 's/jessie/vivid/' -i /etc/apt/sources.list.d/ethereum-*.list
        sudo apt-get update
        ```

        * 安装`ethereum, ethminer 和 geth`:
        ```
        sudo apt-get install ethereum ethminer geth
        ```
        > `geth`好像是用来生成以太币钱包地址的，对于我们将要挖ETP来说，应该没有用处。因为我们在安装好元界钱包，注册登录后会有ETP的地址。

        * 安装显卡驱动:
        首先，需要知道自己电脑的显卡型号，然后去官方下载对应的驱动软件。比如我的显卡是NVIDIA，电脑系统是Linux(Ubuntu 16.04 LTS)64位，使用的是GeForce 920。到[N卡驱动官方网站](https://www.geforce.com/drivers)搜索下载自己需要的驱动。

        然后，安装显卡驱动所需要的依赖：
        ```
        sudo apt-get install linux-headers-amd64 build-essential
        ```
        > `linux-headers-amd64` 这个包好像已经废弃了，不过不影响后续安装。

        最后，安装驱动。显卡驱动安装必须在Linux文本模式，所以：
            第一步，`Ctrl+Alt+F1`切换到`tty1`，使用命令`sudo service lightdm stop`关闭 `X-Window`.
            第二步，给下载的驱动增加执行权限，然后运行(*注：请先看完以下内容在运行*).
            ```
            chmod +x NVIDIA-Linux-x86_64-367.35.run
            sudo ./NVIDIA-Linux-x86_64-367.35.run
            ```
            第三步，安装完成后，重新启动`X-Window`：`sudo service lightdm start`，然后`Ctrl+Alt+F7`进入图形界面；
            我在安装完成后遇到一个坑，在图形模式下，登录界面输入密码后依然跳转回登陆界面，无限循环。经过搜索，用以下方法重新安装解决:
            ```
            sudo ./NVIDIA.run -no-x-check -no-nouveau-check -no-opengl-files
            -no-x-check：安装驱动时关闭X服务
            -no-nouveau-check：安装驱动时禁用nouveau
            -no-opengl-files：只安装驱动文件，不安装OpenGL文件
            ```
            这样再reboot，就不会出现循环登录的问题。
            如果没有解决请参考[这篇blog](http://blog.csdn.net/chaihuimin/article/details/71006654?locationNum=2&fps=1).

	3. 寻找矿池挖矿。
		* [火池](http://etp.huopool.com/)
		> 我使用的是火池，挖矿命令：`ethminer -F http://get.etp.huopool.com:8888/MEEihkdp6w7JKVA6hyKVGU9FomAV4G7jYP -G --farm-recheck 200`

        由于我用的是个人电脑，所以算力很低:)，仅为了试验一下而已，下面是挖矿和矿池收益截图：
        ![挖矿图](http://blog.blianb.com/wp-content/uploads/2017/08/miner.png)
        ![收益图](http://blog.blianb.com/wp-content/uploads/2017/08/huopool-1.png)
        * [双优](http://uupool.cn/)