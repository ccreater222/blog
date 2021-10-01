categories:
- 做题笔记
tags:
- misc
- ctf
title: 某misc
---
# 不知道哪里的misc



题目给了一个压缩包



里面有一个加密的视频文件和未加密的图片

![image105](https://i.loli.net/2020/02/19/ptInrs6axclk12u.png)



想着图片会不会有提示,用hxd打开发现末尾果然有提示

kobe code

![image211](https://i.loli.net/2020/02/19/mVqDzgoHdyOWBnT.png)



得到密码:`OAEBEYTKNRBANB`,成功拿到视频文件



但是提示文件无法打开,用binwalk啥的弄了一会,发现里面没有藏啥

于是换了个思路,猜测这题是要让我们修复mp4文件?头大



>一个MP4文件首先会有且只有一个“ftyp”类型的box，作为MP4格式的标志并包含关于文件的一些信息；之后会有且只有一个“moov”类型的box（Movie Box），它是一种container box，子box包含了媒体的metadata信息；MP4文件的媒体数据包含在“mdat”类型的box（Midia Data Box）中，该类型的box也是container box，可以有很多个，也可以没有（当媒体数据全部引用其他文件时），媒体数据的结构由metadata进行描述。

![image622](https://i.loli.net/2020/02/19/NVA3PtERXBqfwcd.png)



图中方框即为第一个box的head,原本应该是`ftyp(0x66747970)`而这里确实`66459707`,位置反了

恢复脚本

```python
with open('NBA.mp4', 'rb') as f:
    keydata =f.read()
t= keydata.encode("hex")
l=len(t)
t2=""
for i in range(0,l,2):
	a=t[i]
	b=t[i+1]
	t2+=b
	t2+=a

print t2[:100]
t2=t2.decode("hex")
f2 = open('NBA2.mp4', 'wb')
f2.write(t2)

```

等脚本跑完后成功恢复原来的视频文件

视频隐写通常把内容存储在视频中的某个图片,我们用 ffmpeg ,将视频分离成图片

` ffmpeg -i NBA2.mp4 -f image2 image%d.jpg `

一张一张看过去,我们发现,在第3028张突然全黑了

![image1136](https://i.loli.net/2020/02/20/Q9dsOuiVGFgwAcm.png)



调整一下图片的对比对,跑出一个二维码

![image1223](https://i.loli.net/2020/02/20/aE3PpqcrJzn852V.png)



`flag{I_l0v3_pl4ying_b4sk3tb4ll}`

