* ##### 用户访问

> 首先会输入网站URL，例如:http://blog.blianb.com，这个时候DNS( Domain Name System)：“域名系统”，会把域名翻译为对应的IP地址。为什么会有这一步呢？你可以先简单的理解为手机里保存的手机号与对应姓名，QQ号与备注的关系，目的是为了辨识和使用。

1. 输入URL回车后，计算机会先查找浏览器的缓存，如果浏览器没有缓存这个URL对应的IP地址，或者缓存已经过期，那么计算机会接着查找操作系统本地DNS解析器的缓存。
> 注：我们怎么查看Chrome自身的缓存？
>> 可以使用 chrome://net-internals/#dns 来进行查看，如下图，标红为过期:
![浏览器缓存](http://blog.blianb.com/wp-content/uploads/2016/12/dns.png)

2. 计算机检查本地hosts文件里是否含有这个映射关系
Windows的hosts文件一般在C:\Windows\System32\drivers\etc\hosts
Linux的hosts文件一般在/etc/hosts里
> 注:Linux文件路径和内容如下图:
![host文件路径](http://blog.blianb.com/wp-content/uploads/2016/12/hosts_path.png)
![host文件内容](http://blog.blianb.com/wp-content/uploads/2016/12/hosts_content.png)
btw，一般做网站开发的都会修改这个文件，配合本地的web server实现虚拟主机，方便在本地测试，127.0.0.1代表请求本地的服务

3. 查找网络设置里的DNS服务器(区域)

4. 检查此DNS服务器里是否含有这个IP地址映射关系

5. 最后转至根DNS服务器查询，以下是DNS服务器的一些相关资料:

名称类型 | 说明 | 示例 |
:-----------:|:--------------:|:---------------:|
根域 |DNS域名中使用时，规定由尾部句点(.)来指定名称位于根或更高级别的域层次结构| 单个句点(.)或句点用于末尾的名称
顶级域 | 用来指示某个国家/地区或组织使用的名称的类型名称 | .com
第二层域 | 个人或组织在Internet上使用的注册名称 | blianb.com
子域 | 已注册的二级域名派生的域名，通俗的讲就是网站名 | www.blianb.com
主机名 | 通常情况下，DNS的域名的最左侧的标签标识网络上的特定计算机，如h1 | blog.blianb.com

DNS域名称 | 组织类型 |
:---------:|:---------:|
com | 商业公司
edu | 教育机构
net | 网络公司
gov | 非军事政府机构
Mil | 军事政府机构
xx | 国家/地区代码(cn代表中国)

4. DNS工作简图:
![DNS工作简图](http://img1.51cto.com/attachment/201203/175333937.jpg)

------

* ##### 发起TCP三次请求
> 拿到域名对应的IP地址之后，User-Agent（一般是指浏览器）会以一个随机端口（1024 < 端口 < 65535）向服务器的web server（常用的有apache,nginx等）80端口发起TCP的连接请求。这个连接请求（原始的http请求经过TCP/IP4层模型的层层封包）到达服务器端后（这中间通过各种路由设备，局域网内除外），进入到网卡，然后是进入到内核的TCP/IP协议栈（用于识别该连接请求，解封包，一层一层的剥开），还有可能要经过Netfilter防火墙（属于内核的模块）的过滤，最终到达web server（本文就以Nginx为例），最终建立了TCP/IP的连接。

1. 使用子网掩码判断客户机和服务器是否在一个子网络

2. 在同一个子网络，则使用以太网进行广播发送数据包

3. 不在同一个子网络，则通过网关转发

> 涉及的相关资料:
OSI模型七层协议
tcp的三次握手，建立连接
TCP/IP模型是一系列网络协议的总称,TCP/IP模型一共包括几百种协议，对互联网上交换信息的各个方面都做了规定

------

* ##### web server(nginx)
###### 要知道http请求到达web server之后，web server是怎么工作的，就要先知道以下几个概念:
1. CGI：CGI 是 Web Server 与后台语言交互的协议;

2. FPM (FastCGI Process Manager)，它是 FastCGI 的实现，任何实现了 FastCGI 协议的 Web Server 都能够与之通信;

3. FPM 是一个 PHP 进程管理器，包含`master`和`worker`两种进程：`master`进程只有一个，负责监听端口，接收来自`Web Server`的请求，而`worker`进程则一般有多个 (具体数量根据实际需要配置)，每个进程内部都嵌入了一个 PHP 解释器，是 PHP 代码真正执行的地方，下图是我本机上 fpm 的进程情况，1个`master`进程，多个`worker`进程(本地有3个，云服务器上有5个`worker`);
本机:
![本机](http://blog.blianb.com/wp-content/uploads/2016/12/local_fpm.png)
运程服务器：
![远程服务器](http://blog.blianb.com/wp-content/uploads/2016/12/remote_fpm.png)

4. 关于`php-fpm worker`进程的配置说明,详见[php-fpm里进程管理配置介绍](http://wp.me/p87BY0-Kn);

------

* ##### web server按端口监听到请求后，会分配给worker进程，调用相关PHP脚本进行具体的处理。

------

* ##### php-fpm执行php文件
涉及到解释器的执行过程
通过socket和数据库或是缓存等交互

------

* ##### 返回结果
也是通过网络协议

------

* ##### 浏览器显示