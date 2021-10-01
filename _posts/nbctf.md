date: 2020-04-20
categories:
- 比赛
tags:
- wp
- ctf
title: newbie ctf 2019
---
# newbie ctf 2019

## web

### normal_host

题目描述:

>You can get flag in "normalflag.iwinv.net"

题目给的链接

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g8kogjizorj30ue0gvngf.jpg)



直接访问 flag

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g8kofgp0d7j312909paar.jpg)



输入网站测试发现,会在网址后面加个secret参数

![image-20191103111420354](C:\Users\蔡建斌\AppData\Roaming\Typora\typora-user-images\image-20191103111420354.png)



而flag界面就是通过这个secret来识别是否通过internal.iwinv.net

观察secret发现，每次访问给出的secret都不一样

排除掉算出secret的做法,那么接下来就是要绕过禁用host的限制或者是输入一个网址，让其无法识别出是`normalflag.iwinv.net`却能访问`normalflag.iwinv.net`

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g8kolppcsfj30g3045748.jpg)



这就很容易想到php里面libcurl和parase的差异

>parse_url与libcurl对与url的解析差异可能导致ssrf当url中有多个@符号时，parse_url中获取的host是最后一个@符号后面的host，而libcurl则是获取的第一个@符号之后的。因此当代码对http://user@eval.com:80@baidu.com进行解析时，PHP获取的host是baidu.com是允许访问的域名，而最后调用libcurl进行请求时则是请求的eval.com域名，可以造成ssrf绕过此外对于https://evil@baidu.com这样的域名进行解析时,php获取的host是evil@baidu.com，但是libcurl获取的host却是evil.com 
>
>from  https://paper.seebug.org/561 



构造url:`user@4399.com@normalflag.iwinv.net`(我是来帮4399打广告的:v:)​



KorNewbie{H0$7_$P1it_A774cK_U$3s_N0RM^liZ47ioN&##$%%!}



## misc

### catch me



利用工具逐帧查看，把每个子出现的位置记录转成ascii码得到flag



KorNewbie{w0w_e4g1e_3y3}



### BiMilCode

![image.png](http://ww1.sinaimg.cn/large/006pWR9aly1g8kovw3cl4j30e704pt8u.jpg)



刚开始题目说要encode。结果被这个误解搞了一个小时才发现是decode

观察发现一个字符会被编码成2位的16进制的数据

认真观察发现编码方式还和位置有关，但是相同的位置编码方式相同,而且只是对ascii码进行移位

也就是说,在同一个会话里,我们可以通过ascii码的差值来得到来计算出题目给出的编码后的字符串

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g8kp8m10vej30ep058jrl.jpg)



```python
s2='45 dd 97 38 62 6c 62 85'.split(' ')
s1='3a 93 78 3a 6c 63 3a 63'.split(' ')
print(s1)
print(s2)
for i in range(len(s1)):
    print(chr(ord('0')+int(s1[i],16)-int(s2[i],16)),end='')
```





KorNewbie{Nace_I_believed_it}



## Forensic

### find the plain

题目描述

>Alpha team's whistleblower captured a packet that leaked internal information to the outside using ftp internal confidential data.
>
>Analyze the packet and flag the password and information obtained for the ftp connection!
>
>flag format : KorNewbie{password_yougetinformation}
>
>※ If there is a hash value, it will be md5.





用`(!frame.len== 1518) &&(!frame.len== 1514)&& (! ip.addr==192.229.232.240) && (not tcp.stream==2) && (not tcp.stream==0)&& (not tcp.stream==1)`

过滤找到ftp传输文件和账号密码

password:root

badguy.txt

```
7J2067O06rKMIOyVjO2MjO2MgOydmCDsi6Dsg4HsoJXrs7TripQg67CR7J2YIOyjvOyGjOyXkCDrqqjrkZAg64u07JWE64aT7JWY64SkLiDqsbTtiKzrpbwg67mM7KeA7JuM7YSwLi4gDQpodHRwczovL3Bhc3RlYmluLmNvbS83MHlER2lSUw==
```





decode得到

```
이보게 알파팀의 신상정보는 밑의 주소에 모두 담아놓았네. 건투를 빌지워터.. 
//alpha组的个人信息全部放在下面的地址里。祝你成功..
https://pastebin.com/70yDGiRS
```

`k459iki6m5j094m2lmkhjmi9527l81ml`

题目提示得到的一个信息是md5

所以对这个进行凯撒密码解密得到`d459bdb6f5c094f2efdacfb9527e81fe`

后面发现![image.png](http://ww1.sinaimg.cn/large/006pWR9aly1g8k0vd6qapj30ue015gli.jpg)

这里也提示了凯撒加密

但是用国内的md5解密网站无论如何也破不了

用朝鲜语搜了一下就解密了

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g8k0ww99j0j30j6036t8w.jpg)

KorNewbie{root_IronDragon}

...........................

这是个梗吗.

### chat

题目描述

> Confidential information came and went through chat. Find the email of the user who used the chat. 

user:NewbieCTF2019



在youtube找到一个好玩的重置密码的方法

 https://www.youtube.com/watch?v=YBFAzK9mgNI 

根据这个来重置密码

重置密码后看着棒子文我是懵逼的(同伴+1)

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g8kph085tzj31gw0see6k.jpg)

根据题目提示聊天软件理所当然的盯上了kakao

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g8kppnap1sj30wj0b60u7.jpg)



跑到文件夹里翻一翻(也可以写个脚本/fad)

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g8kpqxb2njj30tz0j8my5.jpg)



最后拿到flag



`KorNewbie{renek@it-simple.net}`

### rec

这题跟进去调试也没看到啥，想来想去防止取证题的地方怎么也不可能是逆向啥的、

于是就开始用binwalk,strings.....

最后翻一翻找到了seg段............

在seg段找到

![image.png](http://ww1.sinaimg.cn/large/006pWR9aly1g8k2xe25wlj30io0gbdio.jpg)