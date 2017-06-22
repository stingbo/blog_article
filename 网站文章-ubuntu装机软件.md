###### 个人目前使用ubuntu，主要是用来进行web开发，在安装完ubuntu后，可以看到默认是有浏览器firefox，办公套件LiBreOffice，电邮客户端Thunderbird Mail，好了接下来说说我为了个人需要安装一些软件，仅供参考。

* 通迅相关

> 1. QQ International 这个不是官方发布的客户端，具体开发者我也不知道是谁，但我在使用ubuntu 13.x 的时候就使用的这个客户端，到目前使用的16.04LTS版都没有出现过问题，应该具备QQ2013的所有功能，办公使用足够了。如果有人需要这个deb包可以留言回复，我会发送文件到你邮箱并协助你安装。注意：需要先安装Wine(`sudo apt install wine`).

> 2. Electronic WeChat 微信客户端，这个也不是腾讯官方发布的客户端，是一个开源项目。具体安装方法在[github](https://github.com/geeeeeeeeek/electronic-wechat)上面写的非常清楚，当然，如果你有不明白的地方，也可以回复我，我会尽我所能帮助你。

---
* 输入法

> 1. fcitx ubuntu有自带的ibus，不过我更喜欢使用fcitx，使用它的五笔拼音，打字嗖嗖的。

---
* 浏览器

> 1. google-chrome-stable[安装文档](http://wp.me/p87BY0-Le)

> 2. chromium chromium是开发版，各种新功能都会有，但ubuntu的包却是很落后，刚开始我用的是chromium，后台更换为了chrome。

---
* 音乐

> 1. 厉害了word我的网易，网易云音乐有专门ubuntu对应版本的包，一般人我不告诉他，16.04LTS亲测能用，一直使用的[网易云音乐](http://music.163.com/#/download)。

---
* 虚拟机

> 1. VirtualBox 不管是为了在Linux上使用Windows(Linuxer的痛)还是为了统一开发环境使用vagrant，都需要[安装](http://note.youdao.com/noteshare?id=a70dbdab37ee00f47d6b3d10e0332e56)VirtualBox。

---
* 数据库客户端

> 1. emma 最开始我一直使用的emma，界面简洁，查看方便，能使用SQL语句，相对功能会少一些，且apt安装后有个小问题，不支持中文显示，[解决方法](http://wp.me/p87BY0-i7)。

> 2. navicat for mysql navicat对比其它Linux下Mysql客户端确实强大太多了，常用的各种功能自不在话下，还支持视图，函数，事件等等，[下载](https://www.navicat.com.cn/download)解压后进入目录，运行脚本启动即可，但navicat也有个小问题，它不是免费的，只有15天试用期，不过你可以到目录.navicat下把user.reg删掉(`cd ~/.navicat/`)，然后重启，你就又有15天的试用期了。

---
* markdown编辑器

> 1. Haroopad 界面美观，清晰，支持实时预览等很多功能，还有我最舒适的vim操作模式，我只使用了它的markdown语法编辑功能，目前只能导出HTML格式文件，[下载](http://pad.haroopress.com/user.html)后点击安装即可，此文就是通过这个软件编写的。

> 2. Atom A hackable text editor for the 21st Century，Atom编辑器功能更强大，放到这个模块下面，其实是不合理的，因为它不仅仅是一个markdown编辑器，更是一个功能强大的文本编辑器，由于我使用的是(G)vim，所以只是把Atom当作写markdown文档的工具在使用，使用Atom的原因是家里电脑使用的是ArchLinux，ArchLinux上没有Haroopda的包，不得已找到了Atom，才发现它也非常强大，有超多第三方插件可供选择使用，除了自带插件，我搜索下载了支持vim的vim-mode，md文件转换为pdf的markdown-pdf，此次更新文件使用Atom修改。

---
* 图片编辑工具

> 1. GIMP Image Editor 是一个很强大的图片编辑工具，据说是Linux下的PhotoShop，当时我需要制作一个自己博客的logo，此[网站](http://blog.blianb.com/)的logo就是使用这个软件制作的，最后就找到了这个软件，目前使用了此软件很小一部分功能。

> 2. gnome-screeshot 顺带说一下gnome自带的截图工具，我自己在系统设置，键盘，快捷方式里设置了它的快捷键`ctrl + alt + a`对应`gnome-screenshot -a`，此后截图就非常方便了。

---
* 开发相关

> 1. PHP7 Nginx mariaDB就不用说了.

> 2. vim-gtk 我之前使用的vim-gnome编辑器，后来在每次关闭vim时，都会提示一个错误，在搜索这个问题时，看到Stack Overflow上别人推荐使用vim-gtk，使用下来一点不差.

> 3. Composer PHP 的一个依赖管理工具，[安装文档](http://docs.phpcomposer.com/00-intro.html).

---
* 科学上网

> 1. Shadowsocks科学上网工具，结合[浏览器插件SwitchyOmega](http://wp.me/p87BY0-19)或是[终端代理工具ProxyChains](http://wp.me/p87BY0-1f).注意：直接apt install shadowsocks安装的可能不支持rc-md5，可先安装pip(`sudo apt install python-pip`)，然后用pip安装(`pip install shadowsocks`).

---
* 系统设置

> 1. 使用monaco字体，参考[github一篇文档](https://github.com/cstrap/monaco-font)，clone或下载文件，请仔细阅读README.md，最后找到其中为Ubuntu定制的安装脚本。运行命令`./install-font-ubuntu.sh https://github.com/todylu/monaco.ttf/blob/master/monaco.ttf?raw=true`，文档里提供的地址已失效。

> 2. Unity Tweak Tool设置系统工具，可以在Ubuntu应用商店搜索安装，安装后可以使用第一步里安装好的monaco字体.

---
* 其它相关问题

> 1. 配置thunderbird,qq企业邮箱服务时要补全为exmali.qq.com

> 2. `门`字的问题，一些字体显示不正确。
在 `/ect/honts/conf.d/64-language-selector-prefer.conf` 有以下内容：
```
<alias>
      <family>sans-serif</family>
      <prefer>
         <family>Noto Sans CJK JP</family>
         <family>Noto Sans CJK SC</family>
         <family>Noto Sans CJK TC</family>
      </prefer>
   </alias>
   <alias>
      <family>monospace</family>
      <prefer>
         <family>Noto Sans Mono CJK JP</family>
         <family>Noto Sans Mono CJK SC</family>
         <family>Noto Sans Mono CJK TC</family>
      </prefer>
   </alias>
```
你要做的就是把 JP 换到最后面，然后重启电脑。