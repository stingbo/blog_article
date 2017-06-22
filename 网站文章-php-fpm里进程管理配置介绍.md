* 强调一个观点，给自己的备忘录:
> 关于开发时使用到的相关软件(`php`,`nginx`,`php-fpm`,`mysql`等)的使用信息，最好的方式是查看他们的文档和各自的配置信息.

###### 当然没有一定的知识，可能看了也会云里雾里，在此给出自己对php-fpm配置文件的一些理解.

* 首先，需要知道php-fpm配置文件所在的路径，使用命令`ps -ef | grep php`，如下图:
![本机](http://blog.blianb.com/wp-content/uploads/2016/12/local_fpm.png)
![远程服务器](http://blog.blianb.com/wp-content/uploads/2016/12/remote_fpm.png)
1. 可以看到在两台机器上，配置文件的路径不一样，上图的路径是`/etc/php/5.6/fpm/php-fpm.conf`，下图的路径是`/etc/php-fpm.conf`，而且php-fpm启动的进程数也不一样，`master`进程数都为1，`worker`进程数上图为3个，下图为5个.
2. 配置文件路径不一样是因为软件安装时软件包里指定的路径不一样，但为什么启动的进程数也不一样呢？这就要看配置文件的详细内容了.

* 打开php-fpm.conf，内容如下，详细信息请看注释:

```````````````````````````````````````````````````````````````
;;;;;;;;;;;;;;;;;;;;;
; FPM Configuration ;
;;;;;;;;;;;;;;;;;;;;;

; All relative paths in this configuration file are relative to PHP's install
; prefix (/usr). This prefix can be dynamically changed by using the
; '-p' argument from the command line.

; Include one or more files. If glob(3) exists, it is used to include a bunch of
; files from a glob(3) pattern. This directive can be used everywhere in the
; file.
; Relative path can also be used. They will be prefixed by:
; - the global prefix if it's been set (-p argument)
; - /usr otherwise
; 可以看到，在文件的开始就又加载了另外一个目录下以 .conf 结尾的配置，关于进程运行方式和数量的配置就在这个目录下.
include=/etc/php/5.6/fpm/pool.d/*.conf

........ 以下内容省略，因为此次内容是关于php-fpm进程相关的配置，故只截取了部分内容 ........
```````````````````````````````````````````````````````````````

* 打开对应目录`/etc/php/5.6/fpm/pool.d/`，可以看到`www.conf`文件，打开文件，可以看到内容有很多，我只截取了与进程数相关的内容，大概从74行开始，不同机器，不同版本`php-fpm`，文件内容应该大同小异，详情请看注释

```````````````````````````````````````````````````````````````
; Choose how the process manager will control the number of child processes.
; 注：以何种方式运行php-fpm，进而管理子进程数量，这里列出了三种可选方式:
; 静态模式(static)，动态模式(dynamic)，按需加载模式(ondemand).

; Possible Values:
; static - a fixed number (pm.max_children) of child processes;
; 注：静态模式，这种模式下，进程数是固定的，固定的个数就是参数 max_children 所设置的个数，此个数不可以动态调整，对于个人本地开发或是小站来说，此模式可以节约资源.

; dynamic - the number of child processes are set dynamically based on the
; following directives. With this process management, there will be
; always at least 1 children.
; 注：动态模式，这种模式下，进程数是会动态改变的，但最少必须保留一个进程，它的启始进程数和动态改变最大进程数受以下参数控制.
; pm.max_children - the maximum number of children that can
; be alive at the same time.
; 最大进程数：动态增加时，进程数的最大值，即使 php-fpm 进程数被使用完要报错了，进程数的总量也不会超过这个值的限定.
;
; pm.start_servers - the number of children created on startup.
; 启始进程数：望文生义即可.

; pm.min_spare_servers - the minimum number of children in 'idle'
; state (waiting to process). If the number
; of 'idle' processes is less than this
; number then some children will be created.
; 最少保留进程数：望文生义即可，需要注意的时，即使没有请求需要处理，空闲的进程数少于这个设定值时，php-fpm的进程数还是会增加，这样就会浪费机器资源，要根据机器实际情况来调整.
;
; pm.max_spare_servers - the maximum number of children in 'idle'
; state (waiting to process). If the number
; of 'idle' processes is greater than this
; number then some children will be killed.
; 最大保留进程数：望文生义即可，需要注意的时，在没有请求需要处理，空闲的进程数大于这个设定值时，php-fpm的进程数会被kill掉.

; ondemand - no children are created at startup. Children will be forked when
; new requests will connect. The following parameter are used:
; 注：按需加载模式，服务启动时没有进程启动，但在请求到来时，会自动fork进程，这种模式下，进程数是会动态改变的，但最少必须保留一个进程，它的启始进程数和动态改变最大进程数受以下参数控制.
;
; pm.max_children - the maximum number of children that
; can be alive at the same time.
; 最大进程数：同上.
;
; pm.process_idle_timeout - The number of seconds after which
; an idle process will be killed.
; 进程空闲保留时间(秒)：当进程空闲时，如果在设置的时间内没有被使用将会被kill掉.
; Note: This value is mandatory.
; 注：此参数不能缺省.
pm = dynamic

; The number of child processes to be created when pm is set to 'static' and the
; maximum number of child processes when pm is set to 'dynamic' or 'ondemand'.
; This value sets the limit on the number of simultaneous requests that will be
; served. Equivalent to the ApacheMaxClients directive with mpm_prefork.
; Equivalent to the PHP_FCGI_CHILDREN environment variable in the original PHP
; CGI. The below defaults are based on a server without much resources. Don't
; forget to tweak pm.* to fit your needs.
; Note: Used when pm is set to 'static', 'dynamic' or 'ondemand'
; Note: This value is mandatory.
; 注：非缺省值，含义见上.
pm.max_children = 5

; The number of child processes created on startup.
; Note: Used only when pm is set to 'dynamic'
; Default Value: min_spare_servers + (max_spare_servers - min_spare_servers) / 2
; 注：只能作用于动态模式，以动态模式运行时可为缺省值，缺省值为：最少保留进程数 + (最大保留进程数 - 最少保留进程数) / 2. 含义见上.
; 测试过，如果这个值随便填写重启php-fpm报错，但根据这个公式且不超过max_children，则没有问题，更详细需要进一步验证.
pm.start_servers = 2

; The desired minimum number of idle server processes.
; Note: Used only when pm is set to 'dynamic'
; Note: Mandatory when pm is set to 'dynamic'
; 注：只能作用于动态模式，以动态模式运行时为非缺省值，含义见上.
pm.min_spare_servers = 1

; The desired maximum number of idle server processes.
; Note: Used only when pm is set to 'dynamic'
; Note: Mandatory when pm is set to 'dynamic'
; 注：只能作用于动态模式，以动态模式运行时为非缺省值，含义见上.
pm.max_spare_servers = 3

; The number of seconds after which an idle process will be killed.
; Note: Used only when pm is set to 'ondemand'
; Default Value: 10s
; 注：只能作用于按需加载模式，缺省值为10s.
;pm.process_idle_timeout = 10s;
```````````````````````````````````````````````````````````````

* 总结:

1. 以何种方式运行php-fpm，进而管理子进程数量，这里列出了三种可选方式:
静态模式(static)，动态模式(dynamic)，按需加载模式(ondemand).

2. 不同模式有不同参数控制进程数.

##### 懂了配置参数的含义，就明白在不同的情况下要选择什么运行模式，以及为什么在查看进程数不同的机器上会有所不同了.