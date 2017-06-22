为了`科学上网`，一直找别人提供的`shadowsocks`服务(以下简称`ss`)，使用过免费的，也使用过收费的，免费的要么流量和速度受限，要么不稳定。收费的更坑，说不定什么时候跑路了，我就碰到过。

所以决定自己搭建一个`ss`服务，所以开始在网上查资料，中间经过了这样的几步：

1. 需要一台自己的境外vps(Virtual Private Server 虚拟专用服务器)，注意是境外的哟，因为我们的请求会转发到这台服务器，再转发出去，如果这台服务器就在GFW内，到头来还是在墙内打转。可以购买的vps的地方很多，比如阿里云的香港服务器，AWS的服务器，或是其它第三方的(`Vultr`,`搬瓦工`等等)
2. 在这台服务器上安装`ss`服务，加以配置
3. 在本地安装`ss`客户端，转接自己搭建的服务

--------------------------

第一步:为了最省钱，你可以找那种免费提供试用的vps，我去很多vps的官网上找了，都没有找到相关的活动与优惠，不过，每个提供商会不定期提供，所以，也许是我找的时机不正确，最后我发现了AWS现在(`2017-05-08`)正好有一个活动，可以试用12个月，见下图:
[查看地址](https://aws.amazon.com/cn/console/)
![aws活动](http://blog.blianb.com/wp-content/uploads/2017/05/Screenshot-from-2017-05-08-15-30-35.png)
在注册过程中，也许你之前已经有amazon的账号，那么你可以直接登录，然后做一些验证，最重要的是它需要你绑定信用卡，在这个过程中，我扣了$2美元，也是一顿饭呢，最后就可以创建自己的EC了，虽然我说的很简略，你可能没有看明白，不过不用担心，这个过程其实很简单，去操作吧，有问题欢迎来找我！
> 注意，这里要`ping <Public IP>`一下，把`<Public IP>`换成AWS给你提供的ip地址，检查自己的vps是否能够直接连接，如果ping不通，要先修改自己的安全组(Security groups)，修改为不受限制

第二步:在自己的服务器上安装`ss`服务，因为我创建创建的ec是CentOS，而且AWS好像只提供了7.x的版本，CentOS安装命令：
```
yum install python-setuptools && easy_install pip
pip install shadowsocks
```
也许提示error,no package等信息，只要去网上找到对应的源，添加一下，再安装就好了，很简单，不用担心。
加以配置:
增加配置文件，路径看自己喜好，我增加在家目录下，`～/ss/ssservice.json`
```
{
    "server": "my_server_ip", // 输入本机的 IP 地址，可以使用ifconfig查看自己服务器的ip，不能配置成AWS给你提供的public ip
    "server_port": 8388,
    "local_address": "127.0.0.1",
    "local_port": 1080,
    "password": "mypassword", // 设置密码
    "timeout": 300,
    "method": "aes-256-cfb",
}
```
启动服务：`ssserver -c ~/ss/ssserver.json -d start`

第三步：本地安装`ss`客户端，配置然后连接，这步省略，因为如果你会上面第二步了，这一步就更简单了。