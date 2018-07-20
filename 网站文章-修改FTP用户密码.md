记得ftp账号，但不记得密码怎么办？

由于需要在wordpress站点里安装插件，但是ftp密码忘记了，所以需要修改ftp密码。

1. 切换到root用户下：`sudo su`
2. 更新密码：`passwd bobo(ftp用户名)`，可以先`cat /etc/shadow`查看一下此用户是否存在
3. 提示：
```
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
```

修改完成。