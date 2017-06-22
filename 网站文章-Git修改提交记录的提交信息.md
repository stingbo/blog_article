### Git修改提交记录的提交信息

修改前提交历史记录如下图：
![修改前](http://blog.blianb.com/wp-content/uploads/2017/06/git_commit_message1.png)

修改提交记录的提交信息有两种情况:

* 修改最近一次提交的提交信息
使用命令`git commit --amend`，进入到下图：
![改前](http://blog.blianb.com/wp-content/uploads/2017/06/git_commit_message2.png)
修改提交信息，然后保存退出，
使用命令`git log`，查看历史提交记录，可以发现最近一次提交信息已被修改，如下图：
![修改后提交信息](http://blog.blianb.com/wp-content/uploads/2017/06/git_commit_message3.png)

如果想要修改最近三次提交信息,或者那组提交中的任意一个提交信息呢？那就是我们要说的第二种情况了：

* 修改多个提交信息
使用命令`git rebase -i 节点`，节点即`HEAD~2^`或`HEAD~3`或某一次的提交记录的哈希值;
例：`git rebase -i HEAD~3`进入下图：
![进入rebase -i交互界面](http://blog.blianb.com/wp-content/uploads/2017/06/git_commit_message4.png)

注意：
1. 上面列出的历史提交记录信息正好和我们提交的时间是相反的，我们最近一次的提交反而在最下面；
2. 可以看到，这个文件相当于一个与Git的交互界面，你可以使用不同的命令达到后续不同的操作;
3. 上图中命令在[附录1](#attach)中说明.

第一方法，使用`reword`操作：
如果只需要修改提交历史信息，把`pick`修改为`reword`即可，在列出的记录里，需要修改哪个提交信息，就把这条记录前的`pick`修改为`reword`，保存退出后，Git就会弹出每个提交信息的编辑信息的编辑器，编辑完成退出后，就会弹出下一个，直到全部完成。

第二方法，使用`edit`操作，这个方法能使用所有`git commit --amend`的功能，比如修改提交者的邮箱信息等等：
这里我们只修改倒数第二次的提交信息，把第二个pick修改为edit，如果你要修改这三次所有的提交信息，可以都把pick都修改为edit，如下图：
![修改你要使用的操作命令](http://blog.blianb.com/wp-content/uploads/2017/06/git_commit_message5.png)

保存并退出编辑器后如下图：
![Git提示](http://blog.blianb.com/wp-content/uploads/2017/06/git_commit_message6.png)

接下来，使用命令`git commit --amend`来修改提交信息，你发现了么，是不是和上一种情况很像！
修改提交信息，然后退出编辑器。最后，运行`git rebase --continue`，这个命令将会自动地应用另外两个提交,然后就完成了。
如果需要将不止一处的pick改为edit，需要在每一个修改为edit的提交上重复这些步骤。每一次，Git都将会停止，让你修正提交，然后继续直到完成。

使用命令`git log`，查看历史提交记录，可以发现倒数第二次的提交信息已被修改，如下图：
![修改后提交信息](http://blog.blianb.com/wp-content/uploads/2017/06/git_commit_message7.png)

****
警告：这是一个变基命令-在`HEAD~3..HEAD`范围内的每一个提交都会被重写，无论你是否修改信息。不要涉及任何已经推送到中央服务器的提交-这样做会产生一次变更的两个版本，因而会使他人困惑。

<span id="attach">附录1</span>：

 命令名称   |         说明
:----------:|:-------------------:
   pick | 使用提交，不做修改
   reword | 使用提交，但编辑提交信息
   edit | 使用提交，但会为了使用`amend`命令停止
   squash | 使用提交，但会把列表里后面的提交压缩到第一次合并成一次提交
   fixup | 和使用`squash`类似，但不会保留提交信息
   exec | 使用shell命令，待测试
   drop | 删除掉某一次提交，此次修改将全部消失(危险)
