date: 2020-03-18
categories:
- 杂
tags:
- 工具
title:  docker部署awvs
---
# docker部署awvs



链接: https://pan.baidu.com/s/1QqtdmgOOd-CCPgmD1aZMCw 提取码: buti 

将百度云下载内容放到awvs文件夹下

Dockerfile

```dockerfile
FROM ubuntu:16.04
COPY awvs /awvs
EXPOSE 3443
RUN /awvs/setenv.sh&& echo -e "\nyes\nubuntu\nccreater@163.com\nccr,123456\nccr,123456\n"|/awvs/acunetix_13.0.200217097_x64_.sh && \
cp /awvs/wvsc /home/acunetix/.acunetix/v_200217097/scanner/wvsc &&\
cp /awvs/license_info.json  /home/acunetix/.acunetix/data/license/license_info.json &&\
echo -e '#!/bin/bash\nsu acunetix && (nohup bash ~/.acunetix/start.sh &) && exit'> /etc/rc.local

```

setenv.sh

```shell
#!/bin/bash

shared_obj_deps=(libpango-1.0.so.0 libXext.so.6 libpthread.so.0 libXi.so.6 libgobject-2.0.so.0 libgtk-3.so.0 libdl.so.2 libgdk_pixbuf-2.0.so.0 libX11.so.6 libuuid.so.1 librt.so.1 libexpat.so.1 libglib-2.0.so.0 libXdamage.so.1 libatk-1.0.so.0 libm.so.6 libatspi.so.0 libcups.so.2 libgio-2.0.so.0 libXfixes.so.3 libXrender.so.1 libxcb.so.1 libsmime3.so libcairo.so.2 libXcomposite.so.1 libgdk-3.so.0 libpangocairo-1.0.so.0 libgcc_s.so.1 libX11-xcb.so.1 libdbus-1.so.3 libnss3.so libXrandr.so.2 libnspr4.so libXcursor.so.1 libnssutil3.so libXss.so.1 libasound.so.2 libatk-bridge-2.0.so.0 libc.so.6 libXtst.so.6)
read -d '' source <<EOF
deb-src http://archive.ubuntu.com/ubuntu xenial main restricted #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb http://mirrors.aliyun.com/ubuntu/ xenial multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse #Added by software-properties
deb http://archive.canonical.com/ubuntu xenial partner
deb-src http://archive.canonical.com/ubuntu xenial partner
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-security multiverse
EOF
echo "$source" > /etc/apt/sources.list 
cat /etc/apt/sources.list 
apt-get update
#for i in ${shared_obj_deps[@]}
#do
#	echo install $i
#	apt-get install $i -y
#done
apt-get install libxdamage1 libgtk-3-0 libasound2 libnss3 libxss1 bzip2 sudo libv8-dev -y
echo "install success"

```



将Dockerfile放在awvs同级目录下,打开命令行移动到Dockerfile所在文件夹,`docker build -t awvs .`

用Dockerfile创建镜像后,用` docker run --privileged=true -p 443:3443-it -d awvs "/sbin/init" `来创建容器

接着等待一会就可以运行了,在改一下本机的hosts,美滋滋,记得是用https去访问

![image3135](https://i.loli.net/2020/03/18/5nruJk7RmfLYjtw.png)











## 参考

 [https://youngrichog.github.io/2019/08/10/Docker-AWVS%E6%89%B9%E9%87%8F%E9%83%A8%E7%BD%B2/](https://youngrichog.github.io/2019/08/10/Docker-AWVS批量部署/) 