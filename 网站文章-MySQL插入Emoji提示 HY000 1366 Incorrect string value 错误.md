==本文前提:`mysql  Ver 14.14 Distrib 5.7.14, for Linux (x86_64) using  EditLine wrapper`==

* 问题描述:在获取微信粉丝信息入库时线上环境产生了错误，而在本地开发时是成功的，线上环境提示nickname字段设置了错误字符串值，如下图:
![MySQL错误提示](http://blog.blianb.com/wp-content/uploads/2017/01/Emoji_error.png)

1. 出现这个问题，我明白是有用户的nickname里面含有Emoji导致的.

2. 经过查找资料，我知晓在Emoji字符有一个特殊的地方，在存储时，需要用到4个字节。MySQL中常用的`utf8`字符集的`utf8_general_ci`这个`collate`最大只支持3个字节。所以为了支持能够存储Emoji，需要修改为`utf8mb4`字符集;于是用到了[这篇文章](http:://https://mathiasbynens.be/notes/mysql-utf8mb4#character-sets)里提到的方法，不管是修改`/etc/my.cnf`文件配置，还是在MySQL Client里修改全局配置，错误仍然出现.
> 在查询相关资料时，还知道了一个信息，my.cnf中的default_character_set设置只影响mysql命令连接服务器时的连接字符集，不会对使用libmysqlclient库的应用程序产生任何作用，就是说会影响客户端链接时命令执行时的字符集，不影响我使用php脚本里的使用！[鸟哥相关文章](http://www.laruence.com/2008/01/05/12.html)

3. <span id='img_3'>后来我意识到一个问题，会不会是MySQL的sql_mode造成的问题，之前在做数据库迁移也出现过各种不兼容的问题.</span>
于是，我查看了一下线上MySQL的sql_mode的设置，因为sql_mode是有默认值的，所以在`/etc/my.cnf`里并没有显性写出，所以只能使用命令`select @@sql_mode;`在MySQL的客户端里查看，线上环境如下:
![默认的sql_mode](http://blog.blianb.com/wp-content/uploads/2017/01/defalut_sql_mode.png)

4. 本地环境的如下：
![本地的sql_mode](http://blog.blianb.com/wp-content/uploads/2017/01/local_sql_mode.png)

5. 修改`/etc/my.conf`文件，在mysqld部分增加sql_mode的配置，再次运行程序后成功，修改配置后内容如下图:
![修改后内容](http://blog.blianb.com/wp-content/uploads/2017/01/my_conf.png)

6. 去MySQL官网查看了5.7版本的Server SQL Modes模块这部分的[文档](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_no_engine_substitution)，有这么一句话:
> The default SQL mode in MySQL 5.7 includes these modes: ONLY_FULL_GROUP_BY, STRICT_TRANS_TABLES, NO_ZERO_IN_DATE, NO_ZERO_DATE, ERROR_FOR_DIVISION_BY_ZERO, NO_AUTO_CREATE_USER, and NO_ENGINE_SUBSTITUTION.

	大意是，如果没有覆盖配置，那么你的5.7版本的MySQL的SQL mode默认配置就会是上面提到的那些，[见3图](#img_3)的默认信息。在MySQL官网有对应每部分详细的说明，我怀疑是`STRICT_TRANS_TABLES`模式造成插入不成功，我没有测试，最后我只保留了`NO_ENGINE_SUBSTITUTION`.
> NO_ENGINE_SUBSTITUTION：顾名思义，没有引擎时替换。详细来说就是，在创建或修改表时，当指定的数据表引擎不可用或是未编译时，会替换为MySQL默认设置的存储引擎.
>> 1. 当此项未设置时，创建表和修改表在引擎不可用时它们的表现也不同。创建表时，如果引擎不可用，MySQL会选择默认的存储引擎，并产生一个警告，但表会创建成功;在修改表结构时，如果引擎不可用，也会产生警告，不同的是修改不会生效.
>> 2. 当此项设置时，如果设置的存储引擎不可用，会产生一个错误，并且新建或是修改表都不会成功.