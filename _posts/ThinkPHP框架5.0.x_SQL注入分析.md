categories:
- cve复现
tags:
- cve复现
title: ThinkPHP框架5.0.x_SQL注入分析
---
# ThinkPHP框架5.0.x SQL注入分析 & 数据库配置泄露



## 应用范围

开启debug模式的thinkphp5.0.x



## 漏洞复现

从官网上下载5.0.9完整版

`application/index/controller/index.php`内容如下

```php
<?php
namespace app\index\controller;

use app\model\Gather;

class Index
{
    public function index()
    {
        $ids = input('ids/a');
        $t = db("user");
        $result = $t->where('id', 'in', $ids)->select();

    }
}

```

![image460](https://i.loli.net/2020/02/01/8hmETeuW31p5Zb6.png)



## 漏洞分析

我们在漏洞的关键位置下个断点进行分析,`thinkphp\library\think\db\Builder.php`:383行

调用堆栈为:

![image609](https://i.loli.net/2020/02/01/YFfV74U8cxTLS2H.png)

从index()方法开始

```php
public function index()
    {
        $ids = input('ids/a');
        $t = db("user");
        $result = $t->where('id', 'in', $ids)->select();

    }
```

调用`$t->where('id', 'in', $ids)`返回值的select()方法

我们先看下select()方法,它会将`$option`作为参数调用`Builder::select`方法

```php
public function select($data = null)
    {
    	...
        
        // 分析查询表达式
        $options = $this->parseExpress();

        ...
        if (!$resultSet) {
            // 生成查询SQL
            $sql = $this->builder->select($options);
            // 获取参数绑定
            ...
            }

            ...
        
    }
```

我们跟进`$this->parseExpress();`看`$option`是如何生成的

```php
protected function parseExpress()
    {
        $options = $this->options;
    	...
    }
```

这里是直接令`$options = $this->options;`,而`$this->options`是`$t->where('id', 'in', $ids)`过程生成的

跟进去发现有这一条语句 `$this->options['where'][$logic] = array_merge($this->options['where'][$logic], $where);`,其中`$where`由`$where[$field] = [$op, $condition, isset($param[2]) ? $param[2] : null];`得到

获得`$option`的调用链为:

`$result = $t->where('id', 'in', $ids)->select();` ( `public function where($field, $op = null, $condition = null) ` ) => `$this->parseWhereExp('AND', $field, $op, $condition, $param);`  => `$where[$field] = [$op, $condition, isset($param[2]) ? $param[2] : null];` 



我们回到`$this->builder->select($options);`,现在我们知道`$this->options['where'][$logic]`是我们的可控位置

跟进去

```php
public function select($options = [])
    {
        $sql = str_replace(
            ['%TABLE%', '%DISTINCT%', '%FIELD%', '%JOIN%', '%WHERE%', '%GROUP%', '%HAVING%', '%ORDER%', '%LIMIT%', '%UNION%', '%LOCK%', '%COMMENT%', '%FORCE%'],
            [
                $this->parseTable($options['table'], $options),
                $this->parseDistinct($options['distinct']),
                $this->parseField($options['field'], $options),
                $this->parseJoin($options['join'], $options),
                $this->parseWhere($options['where'], $options),
                $this->parseGroup($options['group']),
                $this->parseHaving($options['having']),
                $this->parseOrder($options['order'], $options),
                $this->parseLimit($options['limit']),
                $this->parseUnion($options['union']),
                $this->parseLock($options['lock']),
                $this->parseComment($options['comment']),
                $this->parseForce($options['force']),
            ], $this->selectSql);
        return $sql;
    }
```

这个`$this->parseWhere($options['where'], $options)`分支进入了关键的漏洞位置,此时`$where`可控

```php
protected function parseWhere($where, $options)
    {
        $whereStr = $this->buildWhere($where, $options);
        ...
        return empty($whereStr) ? '' : ' WHERE ' . $whereStr;
    }
```

再跟进`$this->buildWhere($where, $options)`,此时`$where`可控

```php
public function buildWhere($where, $options)
    {
        if (empty($where)) {
            $where = [];
        }

        if ($where instanceof Query) {
            return $this->buildWhere($where->getOptions('where'), $options);
        }

        $whereStr = '';
        $binds    = $this->query->getFieldsBind($options['table']);
        foreach ($where as $key => $val) {
            $str = [];
            foreach ($val as $field => $value) {
                ...
                    // 对字段使用表达式查询
                    $field = is_string($field) ? $field : '';
                    $str[] = ' ' . $key . ' ' . $this->parseWhereItem($field, $value, $key, $options, $binds);
                
            }

            $whereStr .= empty($whereStr) ? substr(implode(' ', $str), strlen($key) + 1) : implode(' ', $str);
        }

        return $whereStr;
    }
```

`$whereStr .= empty($whereStr) ? substr(implode(' ', $str), strlen($key) + 1) : implode(' ', $str);`这个生成了sql语句的where部分,其中`$str`是由` $str[] = ' ' . $key . ' ' . $this->parseWhereItem($field, $value, $key, $options, $binds);`生成的,而参数`$value[1]`就是我们`ids[1,updatexml(0,concat(0x7e,(database()),0x7e),1)]=123%23`payload中的键值

跟进`$this->parseWhereItem($field, $value, $key, $options, $binds);`

```php
protected function parseWhereItem($field, $val, $rule = '', $options = [], $binds = [], $bindName = null)
    {
    	$key = $field ? $this->parseKey($field, $options) : '';

        // 查询规则和条件
        if (!is_array($val)) {
            $val = ['=', $val];
        }
        list($exp, $value) = $val;
        // 字段分析
        if(....){
            ...
        } elseif (in_array($exp, ['NOT IN', 'IN'])) {
            // IN 查询
            if ($value instanceof \Closure) {
                $whereStr .= $key . ' ' . $exp . ' ' . $this->parseClosure($value);
            } else {
                $value = is_array($value) ? $value : explode(',', $value);
                if (array_key_exists($field, $binds)) {
                    $bind  = [];
                    $array = [];
                    foreach ($value as $k => $v) {
                        if ($this->query->isBind($bindName . '_in_' . $k)) {
                            $bindKey = $bindName . '_in_' . uniqid() . '_' . $k;
                        } else {
                            $bindKey = $bindName . '_in_' . $k;
                        }
                        $bind[$bindKey] = [$v, $bindType];
                        $array[]        = ':' . $bindKey;
                    }
                    $this->query->bind($bind);
                    $zone = implode(',', $array);
                } else {
                    $zone = implode(',', $this->parseValue($value, $field));
                }
                $whereStr .= $key . ' ' . $exp . ' (' . (empty($zone) ? "''" : $zone) . ')';
            }
        } 
        return $whereStr;
    }
```

然后我们可以看到当`in_array($exp, ['NOT IN', 'IN'])`条件满足时,将会直接将我们payload中的键值直接拼接到sql语句中

```php
				foreach ($value as $k => $v) {
                        if ($this->query->isBind($bindName . '_in_' . $k)) {
                            $bindKey = $bindName . '_in_' . uniqid() . '_' . $k;
                        } else {
                            $bindKey = $bindName . '_in_' . $k;
                        }
                        $bind[$bindKey] = [$v, $bindType];
                        $array[]        = ':' . $bindKey;
                    }
```

而`$exp`则由`$result = $t->where('id', 'in', $ids)->select();`中的`'in'`经过一系列操作后得到

但是这里只能进行没有子查询语句的sql报错注入

如果有子查询语句的话就会报`SQLSTATE[HY000]: General error: 1105 Only constant XPATH queries are supported`的错误

所以这个sql注入有点不行,但时sql报错会暴露数据库配置却是非常有用的





## 参考

 https://zerokeeper.com/vul-analysis/thinkphp-framework-50x-sql-injection-analysis.html 

 https://www.leavesongs.com/PENETRATION/thinkphp5-in-sqlinjection.html 

