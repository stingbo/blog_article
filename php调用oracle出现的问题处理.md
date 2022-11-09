环境：
PHP 7.4
框架：laravel6
已安装扩展：oci8-2.2.0，pdo-oci 
数据库：oracle12
已配置了环境变量

使用oci_connect连接，命令行执行：
```
php -r 'var_dump(oci_connect("username", "password", "host:port/database"));'
```
出现问题1：
```
ErrorException
oci_connect(): OCIEnvNlsCreate() failed. There is something wrong with your system - please check that LD_LIBRARY_PATH includes the directory with Oracle Instant Client libraries
```

使用pdo连接，命令行执行：
```
php -r 'var_dump(new PDO("oci:dbname=host:port/database", "username", "password"));'
```
出现问题2：
```
SQLSTATE[HY000]: pdo_oci_handle_factory: Error while trying to retrieve text for error ORA-01804
 (/builddir/build/BUILD/php-7.4.32/ext/pdo_oci/oci_driver.c:714)
```

使用pdo连接，命令行执行：
```
php -r 'var_dump(new PDO("oci:dbname=host:port/database?charset=AL32UTF8", "username", "password"));'
```
问题3：
```
SQLSTATE[HY000]: OCINlsCharSetNameToId: unknown character set name (/builddir/build/BUILD/php-7.4.32/ext/pdo_oci/oci_driver.c:689)
```

上面的问题都是环境变量配置的问题，后来`LD_LIBRARY_PATH`配置对了后，在命令行执行就都好了。

但奇怪的问题出现了，通过浏览器调用接口时，上面的问题又又出现了，我知道是环境变量的问题，也通过网上各种资料尝试，比如直接使用php在代码里设置环境变量，通过 php-fpm 的 www.conf 设置环境变量，结果都没有作用。
有的人在www.conf 里通过下面的配置就解决了问题，但我的没有起作用，如果有人知道原因，麻烦告诉我，谢谢
```
# 都需要修改修改成你自己的路径
env[ORACLE_HOME] = /usr/lib/oracle/12.2/client64
env[LD_LIBRARY_PATH] = /usr/local/lib/instantclient21.8
env[C_INCLUDE_PATH] = /usr/include/oracle/12.2/client64
env[NLS_LANG] = "SIMPLIFIED CHINESE_CHINA.AL32UTF8"
```

由于我的机器是通过 systemctl 来启动php的，经过很多尝试之后，最后在启动php时设定环境来解决了，配置文件`/usr/lib/systemd/system/php74-php-fpm.service`：
```
# It's not recommended to modify this file in-place, because it
# will be overwritten during upgrades.  If you want to customize,
# the best way is to use the "systemctl edit" command.

[Unit]
Description=The PHP FastCGI Process Manager
After=syslog.target network.target

[Service]
Type=notify
ExecStart=/opt/remi/php73/root/usr/sbin/php-fpm --nodaemonize
ExecReload=/bin/kill -USR2 $MAINPID
PrivateTmp=true
# 新增加的环境变量，需要修改成你自己的路径
Environment="LD_LIBRARY_PATH=:/usr/local/lib/instantclient21.8/"

[Install]
WantedBy=multi-user.target
```
