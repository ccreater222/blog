categories:
- 杂
title:   vpn搭建
---
# vpn搭建小记

刚好最近有人叫我帮忙搭个vpn自己去了解了一下,发现搭vpn还挺有意思的



## vpn的最终方案

经过自己瞎折腾后最后是搞成这样的:

家宽-内地中转服务器-gfw-国外vps(使用trojan+bbr)

以后打算在gfw和国外vps在加个cdn,这样我们vps对gfw也就是透明的了,以后gfw把我们vps的ip ban掉了也没关系,不太好的影响就是延迟会变高

这里加个中转服务器是因为电信晚上网络会炸裂,而家宽就是电信,虽然手机是联通的,但是我自己一个月40g流量都不够用



> 电信 163 网连接国际网络，会在高峰时段，在路由出海前的最后一跳，根据优先级，策略性地人为丢包，以减轻对主网的负担(QOS)，这让普通电信用户糟糕的外网访问质量雪上加霜 



## vpn搭建具体细节

### 国外vps选购

首先是vpn的选购,因为是萌新,就去了别人推荐的vultr,搬瓦工太贵了

听说vultr销毁服务器,再重新购买会有新的ip,但是我这样弄还是同一个被ban掉的ip,后来小脑瓜一动,我一口气开了10个服务器,这样ip就不一样了,我真tm机智/cy

具体怎么选购就百度谷歌吧

### 国外vps配置

这里用波仔提供的一键安装脚本,看了一下没啥安全问题

 https://www.v2rayssr.com/trojan-1.html/comment-page-1 

这里你还需要一个域名,建议用二级域名来弄trojan

>1. ` bash <(curl -s -L https://github.com/V2RaySSR/Trojan/raw/master/Trojan.sh) `
>
>2. 先安装bbr,除了重启的选y,其他都选no
>3. 再次运行脚本,运行bbr,并优化配置
>4. 安装trojan
>5. 下载客户端



视频教程: https://www.youtube.com/watch?v=LgZKirXKZms&feature=youtu.be 

### 中转服务器配置

中转服务器去买共享端口+docker虚拟机的那种,一个月5元左右吧,出口最好联通

xxx.sh

```shell
#!/bin/bash
if [ ! $# -eq 3 ];then
    echo -e "Usage : $0 natport trojanserver trojanport"
    exit
fi
natport=$1
trojanserver=$2
trojanport=$3
apt-get install iptables || yum install iptables
iptables -t nat -A PREROUTING -p tcp --dport $natport  -j DNAT --to-destination "$trojanserver:$trojanport"
iptables -t nat -A POSTROUTING -d $trojanserver -p tcp --dport $trojanport -j MASQUERADE
iptables -I FORWARD -d $trojanserver -p tcp --dport  $trojanport -j ACCEPT
iptables -I FORWARD -s $trojanserver  -p tcp --sport  $trojanport -j ACCEPT
service iptables save
service iptables restart
echo "setting complete"
```



### 客户端配置

将config.json拷贝一份,config_through_nat.json

并做出如下修改




```json
{
	...
    "remote_addr": "中转服务器ip",
    "remote_port": 中转的端口,
   ...
    "ssl": {
        "verify": false,
        "verify_hostname": false,
        ...
    }
    ...
}

```

config.json重命名为:config_normal.json

start.bat

```shell
@ECHO OFF
taskkill /im trojan.exe /f
ping -n 2 127.1 >nul
copy /Y config_normal.json config.json
%1 start mshta vbscript:createobject("wscript.shell").run("""%~0"" ::",0)(window.close)&&exit
start /b trojan.exe
```

start_with_nat.bat

```shell
@ECHO OFF
taskkill /im trojan.exe /f
ping -n 2 127.1 >nul
copy /Y config_through_nta.json config.json
%1 start mshta vbscript:createobject("wscript.shell").run("""%~0"" ::",0)(window.close)&&exit
start /b trojan.exe
```

将两个批处理放到trojan客户端目录下



## vpn测试



通过 https://fast.com/  来测速 稳定10Mbits(联通),5Mbits(电信平时),500Kbits(电信夜间,丢包率贼高,体验极差),稳定1Mbits(走中转服务器路线,因为我专门拿来中转的服务器还没买,暂时直接用阿里云的服务器来中转,但是阿里云服务器也在电信机房),去买了个中转的服务器现在夜间电信稳定5Mbits

利用BEST TRACE来看vps的线路

各种线路的分析文章: https://www.duangvps.com/archives/135 

youtube打开4k视频右键查看统计信息,我的是5w左右

linux回程路由测试: https://github.com/nanqinlang-script/testrace 








