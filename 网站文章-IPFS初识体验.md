###### 本示例所用环境

    * 机器:普通电脑 ideapad Y410
    * 系统:ArchLinux

###### 什么是IPFS？ [以下内容来自维基百科自](https://zh.wikipedia.org/zh-cn/%E6%98%9F%E9%99%85%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F)

注意:[IPFS官网](https://ipfs.io/)已经被墙!!!

星际文件系统（InterPlanetary File System，缩写IPFS）是一个旨在创建持久且分布式存储和共享文件的网络传输协议。它是一种内容可寻址的对等超媒体分发协议。在IPFS网络中的节点将构成一个分布式文件系统。它是一个开放源代码项目，自2014年开始由Protocol Labs在开源社区的帮助下发展。其最初由Juan Benet设计。

IPFS是一个对等的分布式文件系统，它尝试为所有计算设备连接同一个文件系统。在某些方面，IPFS类似于万维网，但它也可以被视作一个独立的BitTorrent群、在同一个Git仓库中交换对象。换种说法，IPFS提供了一个高吞吐量、按内容寻址的块存储模型，及与内容相关超链接。这形成了一个广义的Merkle有向无环图（DAG）。IPFS结合了分布式散列表、鼓励块交换和一个自我认证的命名空间。IPFS没有单点故障，并且节点不需要相互信任。分布式内容传递可以节约带宽，和防止HTTP方案可能遇到的DDoS攻击。

该文件系统可以通过多种方式访问，包括FUSE与HTTP。将本地文件添加到IPFS文件系统可使其面向全世界可用。文件表示基于其哈希，因此有利于缓存。文件的分发采用一个基于BitTorrent的协议。其他查看内容的用户也有助于将内容提供给网络上的其他人。IPFS有一个称为IPNS的名称服务，它是一个基于PKI的全局命名空间，用于构筑信任链，这与其他NS兼容，并可以映射DNS、.onion、.bit等到IPNS。

###### 安装
[github提供的几种安装方式](https://github.com/ipfs/go-ipfs#install)

###### 怎么使用
简单使用可以参考这篇博方[如何使用星际文件传输网络（IPFS）搭建区块链服务](http://liyuechun.org/2017/09/18/ipfs-blockchain/)，虽然它是转载[自这](https://qtum.org/zh/blog/ru-he-shi-yong-xing-ji-wen-jian-chuan-shu-wang-luo-ipfs-da-jian-qu-kuai-lian-fu-wu)，但它的格式要好太多:) 。

###### 几点疑问
1. 目前看到的都是用来存储静态文件，感觉和FTP或者CDN有点像，当然也有人在IPFS上写了一个[分布式的点对点聊天应用程序](https://github.com/orbitdb/orbit)。我主要是用PHP来做动态站的，HTTP协议用在客户端与服务端之间的通讯，至于服务端是怎么处理的它并不关心，可是如果用IPFS取代HTTP协议，它怎样处理这种到服务器的请求呢？因为基于IPFS的请求都是返回一个文件，而HTTP请求到服务器后可以通过不同的Web Server进行处理。

2. 今天写了一个小demo，一个静态页面，加上一些js效果，增加到IPFS结点后都能访问，但是当我通过ajax去请求后端脚本时，POST请求直接405错误，GET请求返回的是整个脚本文件的内容，因为它没有通过对应脚本文件的服务去解析它。我在想它到底怎么才能满足这个需求呢？还是说它已经可以满足了而我还不知道？[我的IPFS线上示例](https://ipfs.io/ipfs/QmVbjQfqx3y1sNQGzPRTHqq2aX4pgKEvHnoEbJR4ySTirs)

3. To Be Continue...

***
补一张本地运行webui后的图，感觉这个地球不错!
![webui](http://blog.blianb.com/wp-content/uploads/2018/03/ipfs.jpg)
