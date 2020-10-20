##### 自动化检测PHP语法和编程规范

使用到的知识点：

* 命令`php -l`检测文件语法，可以通过`php -h`查看PHP CLI支持哪些操作，如下:.

```
[:~]$ php -h
Usage: php [options] [-f] <file> [--] [args...]
   php [options] -r <code> [--] [args...]
   php [options] [-B <begin_code>] -R <code> [-E <end_code>] [--] [args...]
   php [options] [-B <begin_code>] -F <file> [-E <end_code>] [--] [args...]
   php [options] -S <addr>:<port> [-t docroot] [router]
   php [options] -- [args...]
   php [options] -a

  -a               Run interactively
  -c <path>|<file> Look for php.ini file in this directory
  -n               No configuration (ini) files will be used
  -d foo[=bar]     Define INI entry foo with value 'bar'
  -e               Generate extended information for debugger/profiler
  -f <file>        Parse and execute <file>.
  -h               This help
  -i               PHP information
  -l               Syntax check only (lint)
  -m               Show compiled in modules
  -r <code>        Run PHP <code> without using script tags <?..?>
  -B <begin_code>  Run PHP <begin_code> before processing input lines
  -R <code>        Run PHP <code> for every input line
  -F <file>        Parse and execute <file> for every input line
  -E <end_code>    Run PHP <end_code> after processing all input lines
  -H               Hide any passed arguments from external tools.
  -S <addr>:<port> Run with built-in web server.
  -t <docroot>     Specify document root <docroot> for built-in web server.
  -s               Output HTML syntax highlighted source.
  -v               Version number
  -w               Output source with stripped comments and whitespace.
  -z <file>        Load Zend extension <file>.

  args...          Arguments passed to script. Use -- args when first argument
                   starts with - or script is read from stdin

  --ini            Show configuration file names

  --rf <name>      Show information about function <name>.
  --rc <name>      Show information about class <name>.
  --re <name>      Show information about extension <name>.
  --rz <name>      Show information about Zend extension <name>.
  --ri <name>      Show configuration for extension <name>.

```
********************
* 安装PHP编程规范工具`php-cs-fixer`，可以在[GitHub](https://github.com/FriendsOfPHP/PHP-CS-Fixer)或[官网](https://cs.symfony.com/)上查看详细信息.
    1. 下载:`wget https://cs.symfony.com/download/php-cs-fixer-v2.phar -O php-cs-fixer`
    2. 给予执行权限:`sudo chmod a+x php-cs-fixer`
    3. 把文件移动到自己喜欢的目录，我一般是放在`/usr/local/bin`下:`sudo mv php-cs-fixer /usr/local/bin/php-cs-fixer`

********************
* 利用Git的钩子`pre-commit`达成commit前自动检测功能.
这个钩子，顾名思意就是在`git commit`之前会触发，此钩子结合网上和朋友的写法，感谢他们.
首先，[点击查看这个文件](./pre-commit)，复制文件内容.
然后，进入到自己的项目下`git`钩子目录:`cd path/to/your/project/.git/hooks`，复制pre-commit.sample文件并重命名:`cp pre-commit.sample pre-commit`，把文件内容替换为上一步复制的内容.
最后，每次commit时就会先检测语法和规范是否正确，不正确会提示文件名和你需要规范代码格式的命令，简单测试如下图：.
![php-cs-fixer pre-commit](http://blog.blianb.com/wp-content/uploads/2017/06/php-cs-fix_pre-commit.png)
