###### 问题原因：
> apt包管理系统具有一组可信密钥，用于对每个包进行身份验证，确定每一个包是否可以信任的安装在系统上。有时，系统没有所有需要的密钥进行验证，所以会遇到这个问题。幸运的是，系统会列出缺失的每个密钥的信息，只需要把这个密码添加到apt密钥管理器中，以便它可以验证包就可以了.


* 使用命令`sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys xxxxxx`
`xxxxxx` 代表系统提示你缺少的密钥，我缺少的密钥是`40976EAF437D05B5`，使用后结果如下图，然后再`sudo apt update`就没有问题了.
![增加密钥](http://blog.blianb.com/wp-content/uploads/2017/03/public_key.png)