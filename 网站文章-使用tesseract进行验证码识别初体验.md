###### 本示例所用环境

    * 机器:普通电脑 ThinkPad E470c
    * 系统:Ubuntu 16.04 LTS

###### 为什么要做验证码自动识别
	你被验证码折磨的还不够惨吗！？:)

###### 什么是ocr和tesseract
    OCR是Optical Character Rcognition的缩写，意思是光学字符识别，针对印刷体字符，采用光学的方式将纸质文档中的文字转换成为黑白点阵的图像文件，并通过识别软件将图像中的文字转换成文本格式，供文字处理软件进一步编辑加工的技术。

    Tesseract是一个跨平台的ocr引擎，适于多种操作系统。它是一款免费软件，根据Apache许可证2.0版发布，当前(2018-03-06)稳定版为3.05.01，自2006年开发以来一直由Google赞助，2006年，Tesseract被认为是当时最准确的开源OCR引擎之一。

###### 怎么使用
    我是在Python脚本里使用，先安装相关库，在Debian / Ubuntu上：

`sudo apt install tesseract-ocr libtesseract-dev libleptonica-dev`

`sudo pip3 install tesseroc`

`sudo pip3 install pillow`

    pillow fork自PIL库，PIL是Python Imaging Library的缩写，Python图片处理库。

代码片段：
```python
import tesserocr
from PIL import Image

print(tesserocr.tesseract_version())
print(tesserocr.get_languages())

image = Image.open('sample.jpg'))
print(tesserocr.image_to_text(image))

print(tesserocr.file_to_text('sample.jpg'))
```

###### 更高级的功能
	以上功能只能识别简单验证码，稍微复杂一点的验证码都识别不出来，更高级的使用则需要对图片进行各种处理，一般是对图片进行二值化、降噪、切割等，然后对Tesseract进行训练，提高识别率，我没有实践过，就不班门弄斧啦！

***
__完结__
