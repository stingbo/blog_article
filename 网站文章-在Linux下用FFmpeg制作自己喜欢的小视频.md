##### 前言

你想在Linux下自由的创作自己的视频吗？快来试试FFmpeg吧！

* 查看自己屏幕分辨率
```
xrandr
```
	注："xrandr" 是一款官方的 RandR Wikipedia:X Window System 扩展配置工具。它可以设置屏幕显示的大小、方向、镜像等。比如屏幕分辨率是1366×768

---

* 屏幕录制
```
ffmpeg -video_size 1366x768 -framerate 25 -f x11grab -i :0.0+0,0 -f pulse -ac 2 -i default `date +%Y-%m-%d_%H:%M:%S`.mp4
```
		 注：如果提示：The encoder 'aac' is experimental but experimental codecs are not enabled, add '-strict -2' if you want to use it.
		 则按提示增加参数'-strict -2'，如:ffmpeg -video_size 1366x768 -framerate 25 -f x11grab -i :0.0+0,0 -f pulse -ac 2 -i default -strict -2 `date +%Y-%m-%d_%H:%M:%S`.mp4

 ---

* 给视频增加或修改背景音乐
```
ffmpeg -i input.mp3 -i output.mp4 [-strict -2](可选) output.mp4
```
	注：如果视频和音频时间不一样长，需要自己额外处理

---

* 视频截取
```
ffmpeg -i input.mp4 -ss 00:00:10 -t 120 output.mp4
```
	注：从第10秒开始，截取120秒

---

* 打马赛克
```
ffmpeg -i input.mp4 -filter_complex "crop=298:80:115:38, boxblur=10[blurLogo1]; [v:0] [blurLogo1]overlay=115:38" -c:a copy -y output.mp4
```
	注：打马赛克，其实是先从原视频里截取一个小区域并模糊处理，产生新视频，然后再把新视频合并回去，这样看起来原视频的某个区域就像打了马赛克一样

---

* 加logo水印
```
ffmpeg -i input.mp4 -i ../watermark.jpg -filter_complex "overlay=main_w-overlay_w-55:5" -codec:a copy output.mp4
```
	注：在overlay的位置加上watermark.jpg图片

---

* 从视频中截图
```
ffmpeg -i input.mp4 -y -f image2 -t 0.001 -ss 10 -s 1280x720 output.jpg
```
	注：在视频的第10秒处生成1280×720的图片

---

* 制作GIF图
```
ffmpeg -ss 10 -t 5 -i input.mp4 -r 10 -vf scale=-1:144 -y output.gif
```
	注：从视频的第10秒开始计时5秒生成GIF图片

---

* 合并视频
```
vim filelist.txt
file 'input1.mp4'
file 'input2.mp4'
file 'input3.mp4'
ffmpeg -f concat -i filelist.txt -c copy output.mp4
```
	注：如果有三个视频文件需要合并，建立一个txt文本，把文件名写入，然后用命令合并