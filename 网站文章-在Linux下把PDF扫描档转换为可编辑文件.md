###### 本教程所用环境

    * 机器:普通电脑 ThinkPad E470c
    * 系统:Ubuntu 16.04 LTS

###### 背景

        有一个朋友找我帮忙，需要把他在网上下载的一份pdf文件转换为word文档。刚开始我认为这个操作很简单，网上应该很多成熟的方案，果然，随便找找，就找到很多免费的在线转换网站。
        本以为事情结束了，native :) ，他下载的是一份扫描档pdf文件(每一页类似于一张文件)，在线网站还没有找到能直接把pdf扫描档转换为word文档的，花了九牛二虎之力也没有发现简便的方法，最后我觉得应该借助Linux的工具来解决这件事(Windows有收费软件直接解决，更方便)。


###### 方法

> 概括:先把pdf扫描档转换为图片，然后把图片转换为txt文本文档，最开始找到了一个软件`pdfocr`，但好像16.04安装不了。

* 把pdf扫描档转换为图片

	这一步需要借助工具包`poppler-utils`，这个工具包里含有很多处理pdf文档的工具。

    1. 检测你是否已经安装过此工具`dpkg -s poppler-utils`，大多数Ubuntu默认已经安装了。如下图：
    ![pdf转换工具](http://blog.blianb.com/wp-content/uploads/2017/09/poppler-utils.png)
    可以看到，有很多命令(`pdftotext`,`pdftohtml`,`pdfimages`等)供我们使用，如果是正常pdf文档是可以直接转换为txt文本的，命令:`pdftotxt xx.pdf xx.txt`。

    2. 如果没有安装，则使用`sudo apt install poppler-utils`命令安装。

    3. 把pdf扫描档转换为png图片：`pdfimages xxx.pdf -png 你的/存放/图片/目录/路径`，如果你的pdf有多页，那么每页会自动生成一张图片。

* 把图片转换成txt文档

	这一步需要借助工具是[Tesseract][tess]，这个软件已经大名顶顶了。

    1. 安装命令`sudo apt install tesseract-ocr tesseract-ocr-chi-sim`，前一个是软件，后一个是中文简体语言包。语言包有很多，我只安装了中文简体，其它语言包如下图:
    ![Tesseract语言包](http://blog.blianb.com/wp-content/uploads/2017/09/tesseract-ocr-lang.png)

    2. cd进入`你的/存放/图片/目录/路径`。

    3. 先处理单个文件试验一下：`tesseract xx.png output -l chi_sim`，参数：`-l(L字母小写,language首字母) chi_sim`是需要识别的语言包，成功后，当前目录下就会多一个output.txt文件，里面内容就是图片上的文字，但要注意，如果图片质量不好，那么识别出来的内容会很偏差。

    4. 批量处理命令:
    ```
    for i in `ls *.png | awk -F '.' '{print $1}'`;do tesseract $i.png $i -l chi_sim;done
    ```

##### ps:如果你有更好的方法，可以微信或邮件告知哟！

[tess]:https://en.wikipedia.org/wiki/Tesseract_(software)