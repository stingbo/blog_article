* 首先，请粗略查看[Thrift安装说明文档](http://thrift.apache.org/docs/install/)，这里列出了不同系统需要的各种依赖，后面编译安装失败的原因有可能是缺少依赖，第一步就是把需要的依赖解决，我的系统是Ubuntu 16.04 LTS，所以参照的是[这里的说明](http://thrift.apache.org/docs/install/debian);

* 其次，[下载thrift源码和示例](http://thrift.apache.org/download)，下载完成后可用md5sum校验文件是否下载完整。解压文件`tar -xzvf thrift-0.10.0.tar.gz`，解压后目录里包含很多文件，有现有可使用thrift的各种语言的库，在`lib`目录里，以及对应语言的使用示例，在`tutorial`目录;

* 然后，配置，编译，安装:
1. `./configure && make`
2. `sudo make install`
3. 检查是否安装成功`thrift -version`，我的显示`Thrift version 0.10.0`，安装成功，版本为0.10.0
> 可参考官方文档[Building from source](http://thrift.apache.org/docs/BuildingFromSource)

* 最后，创建php服务端和用户端，开发流程:
1. 配置环境.PHP的php-fpm,Nginx等.
2. 根据需求，编写thrift接口定义文件(IDL定义文件)，解压的目录里已经包含各种示例所需的IDL文件，在`解压目录/tutorial/`.
3. 使用thrift程序，为不同的语言生成代码.
4. 根据需求，修改生成的代码（主要是Server端），编写实际的业务逻辑.
> [参考博客](http://tutuge.me/2015/04/19/thrift-example-cpp-and-php/?utm_source=tuicool&utm_medium=referral)

* 实际步骤:
1. 移动到PHP项目目录下.
2. 创建目录`mkdir thrift_php_server_client_demo`.
3. 复制上面解压目录PHP相关文件到`thrift_php_server_client_demo`目录下:
> `cp -r 你的解压目录/lib/php/lib/Thrift/ xxx/thrift_php_server_client_demo`
> `cp -r 你的解压目录/tutorial/php/ xxx/thrift_php_server_client_demo`
> `cp 你的解压目录/tutorial/tutorial.thrift xxx/thrift_php_server_client_demo`
> `cp 你的解压目录/tutorial/shared.thrift xxx/thrift_php_server_client_demo`
4. 生成php代码`thrift -r --gen php tutorial.thrift`
> 可参考官方文档[Apache Thrift Tutorial](http://thrift.apache.org/tutorial/)
5. 运行PHP服务端`php -S localhost:8080`
6. 运行客户端`php php/client.php --http`，即可以看到结果运行结果，运行服务端的终端也会显示请求过程.
> 注：我在运行客户端文件时，一直提示server.php里`\tutorial\CalculatorProcessor`类找不到，开始以为是我命名空间或其它配置有问题，后来查看对比了这个[php-thrift-server项目](https://github.com/fisherMartyn/php-thrift-server)才知道，我使用步骤4生成的目录文件(`gen-php/tutorial/Calculator.php`和`gen-php/shared/SharedService.php`)里都缺少相关内容，这个可太坑了，花费了好长时间才发现;我clone了这个项目后，用这个项目里的thrift文件生成代码后也缺少相关类的内容，我怀疑是我安装的thrift版本的问题，把内容加上就正常了。
7. 我的[github示例地址](https://github.com/stingbo/thrift_php_server_client_demo)，下载代码即可使用上述步骤5和步骤6运行.